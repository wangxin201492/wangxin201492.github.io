---
title: MongoDB 源码中定时器机制
date: 2020-05-06 09:42:24
tags:
- MongoDB-OTT
- MongoDB:TaskExecutor
- MongoDB:PeriodicRunner
categories:
- MongoDB
---

MongoDB 提供了2种定时执行机制

- `TaskExecutor::scheduleWorkAt(Date_t when, CallbackFn&& work)`
- `PeriodicRunner::PeriodicJob`

 

## TaskExecutor::scheduleWorkAt

`TaskExecutor::scheduleWorkAt` 提供了一个在指定时间 `when` 执行的方式，具体在 `ThreadPoolTaskExecutor` 中实现。



`ThreadPoolTaskExecutor`实现 `scheduleWorkAt` 方法和 `scheduleWork` 逻辑基本一致：将 `CallbackFn` 添加到一个 `WorkQueue` 中，并获得一个 `CallbackHandle`，随后将 `WorkQueue` 插入到 `_poolInProgressQueue` 中等待被处理。区别在于：

- `scheduleWork` 将 `CallbackFn` 添加到一个 temp 的 WorkQueue，而 `scheduleWorkAt` 添加到 `_sleepersQueue`
- `scheduleWork` 中直接将 `WorkQueue` 中数据插入到 `_poolInProgressQueue` 中，而 `scheduleWorkAt` 则是通过 `_net::setAlarm` 注册一个时间 when 后回调插入的(`_net` 为一个 `NetworkInterfaceTL` 成员)。



 ```c++
StatusWith<TaskExecutor::CallbackHandle> ThreadPoolTaskExecutor::scheduleWorkAt(
                                         Date_t when, CallbackFn&& work) {
    ...
    auto wq = makeSingletonWorkQueue(std::move(work), nullptr, when);
    ...
    auto cbHandle = enqueueCallbackState_inlock(&_sleepersQueue, &wq);
    ...

    auto status = _net->setAlarm(cbHandle.getValue(), 
                                 when, 
                                 [this, cbHandle = cbHandle.getValue()](Status status) {
            ...
            scheduleIntoPool_inlock(&_sleepersQueue, cbState->iter, std::move(lk));
        });
    ...
    return cbHandle;
}
 ```



>  **Note:**   `scheduleWorkAt` 和 `scheduleWork` 都会返回一个 `TaskExecutor::CallbackHandle` 对象  `cbHandle`，这个 `cbHandle` *不是一个回调函数*，应该是表示回调状态 的一个对象， `CallbackHandle` 持有一个 `CallbackState` , 而 `CallbackState` 的类注释为 *"Class representing a  scheduled callback and providing methods for interacting with it."*
>
> 具体的回调函数即上文提到的 `CallbackFn` 被添加到 `WorkQueue`  中，而后被添加到*   `_poolInProgressQueue` 等待被消费



`NetworkInterfaceTL::setAlarm` 接收 3 个入参：`cbHandle`, `when`, `action`，`cbHandle` 主要做一些 状态管理 、 合理性校验等逻辑。action即为当面提到的将 `WorkQueue` 添加到 `_poolInProgressQueue` 的回调。

 

`NetworkInterfaceTL::setAlarm` 中 构造了一个 `PromiseFuture` 对象 **pf ( future为 action）**，并将时间点 when，回调状态 `cbHandle`，新建一个 `ASIOReactorTimer timer`，以及 pf.promise 封装成一个 `AlarmState` 。然后调用 `timer->waitUntil( when )` 等待到达指定时间（witerUntil 实际上也是返回一个 Future），指定时间到达时调用 `_answerAlarm` 标记 满足 promise（及触发上文 action ）

 关于指定时间执行的行为则在 `NetworkInterfaceTL::setAlarm` 中实现。



`ASIOReactorTimer` 中会持有一个 `_timer (asio::system_timer)` ，`waitUntil()` 中会依次调用 `_timer->expires_at()`, `_timer->async_wait()`进行计时等待

 

**总结一下**：`TaskExecutor::scheduleWorkAt` 实际上是基于 `asio::system_timer` 库实现的指定时间执行逻辑。

- 指定时间到达后，会标记 满足 promise ，调用 `_answerAlarm` 
-  `_answerAlarm` 中进行一系列判断后 也会标记 满足 promise ， 调用 action
- action 中 会将 `_sleepersQueue` 中的数据插入到 `_poolInProgressQueue` 排队等待执行



由于 整个逻辑中回调函数比较多，所以堆栈会比较冗长，这里是以 为例的堆栈：

```
#0  mongo::ReplicaSetMonitor::_doScheduledRefresh (this=0x7f0362e0fb00, currentHandle=...) at src/mongo/client/replica_set_monitor.cpp:249
#5  0x00007f035e3fdd28 in mongo::executor::ThreadPoolTaskExecutor::runCallback (this=0x7f0362e19140, cbStateArg=...) at src/mongo/executor/thread_pool_task_executor.cpp:659
#7  0x00007f035e3ff7ad in mongo::executor::ThreadPoolTaskExecutor::scheduleIntoPool_inlock(mongo::executor::ThreadPoolTaskExecutor::WorkQueue*, const iterator&, const iterator&, std::unique_lock<std::mutex>)
#16 0x00007f035e3fee44 in mongo::executor::ThreadPoolTaskExecutor::scheduleWorkAt(mongo::Date_t, mongo::executor::TaskExecutor::CallbackFn&&)
#30 0x00007f035e43d6f2 in mongo::executor::NetworkInterfaceTL::_answerAlarm(mongo::Status, std::shared_ptr<mongo::executor::NetworkInterfaceTL::AlarmState>)
```

 

## PeriodicRunner::PeriodicJob

 Periodic* 类涉及3个接口：

- `PeriodicJob` 描述一个周期性 job，接收3个参数：**job名称 name, Job执行逻辑 callable, 执行间隔 period**
- `ControllableJob : PeriodicJob` 实际执行逻辑，*start / pause / resume / stop* .      
  - `PeriodicJobImpl` 继承实现了该类。
- `PeriodicJobAnchor` : 包装一个 `ControllableJob` ，用于控制 `ControllableJob` 的执行。*start / pause /     resume / stop* .
- `PeriodicRunner` ： 提供一个`makeJob` -- 初始化 `PeriodicJob` / `PeriodicJobImpl` / `PeriodicJobAnchor`.
  - `PeriodicRunnerImpl` 继承实现了该类

 

上面这几个接口主要提供 job 的描述，job 运行的管理，但是也有2个比较奇怪的地方：

- `PeriodicJobImpl` 看名称疑似 `PeriodicJob` 的具体实现，然而却是 `ControllableJob` 的实现
- `PeriodicJobAnchor` 和 `ControllableJob` 都提供了 job 的 *start / pause / resume / stop* 行为。`PeriodicJobAnchor` 中包装一个 `ControllableJob` 对象 _handle，所有的 job 行为也都直接调用了 _handle 对应的函数

 

每个 job 有独立的线程处理，线程的名称即为 job 的名称。job 的运行管理在 `PeriodicJobImpl::_run` 中实现，负责管理的 *pause / resume / stop* 等行为会修改 job 的执行状态，`_run` 在执行前根据被标记的执行状态作出具体的处理。



正常情况下( RUNNING状态 )，_run 会调用 `clockSource->waitForConditionUntil`  ( _clockSource 为 `servicecontext ->getPreciseClockSource()` )，这中间由于 `tracksSystemClock = true / waitable = null`，所以时间等待最终的实现是 `stdx::condition_variable.wait_util(lock, deadline);`

 

> **Note:**   PeriodicRunner::PeriodicJob 虽然进行初始化的时候传入的是一个 interval ，但是实际最终执行还是基于上次  job 开始执行的时间 + interval 获得一个 deadline 来定义下一次任务的触发时