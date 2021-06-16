---
title: MongoDB ReplicaSetMonitor对shard探测机制
date: 2020-04-27 12:59:24
tags:
- MongoDB-OTT
- MongoDB:ReplicaSetMonitor
categories:
- MongoDB
---



sharding实例对后端 shard 会进行状态探测，以发现 shard 是否有节点更新（ 选主、节点加入、节点异常）。提供探测能力的核心类为 `ReplicaSetMonitor` 及相关类。

 

对于状态探测的基本逻辑应该也比较好构思：**定期状态探测、探测结果存储、探测结果查询**。MongoDB中也按照这个逻辑来完成的： 

- `ReplicaSetMonitor` 提供状态查询的接口，以及定期探测触发能力。状态信息存储在 `_state` (`SetState`类型成员变量) 中

- `SetState` 存储 ReplicaSet shard 的状态信息。包装 Node 记录各个 Node 的信息，持有一个 `ScanState` 记录该 ReplicaSet 当前正在 scan 的信息

- Refresher 进行实际的状态探测，提供一个 static 方法 `ensureScanInProgress` 来初始化一个 Refresher 进行探测。

  - `Refresher` 与 `SetState` 是一一绑定的，只有 `SetState` 在被探测状态，才会有对应的 `Refresher` 产生

- `ScanState` 记录 `Refresher` 探测过程中的状态信息。`ScanState` 会被 `SetState::currentScan` 及 `Refresher::_scan` 持有，一般情况下这2个 scan 是相同的，但是可能存在并发的场景导致指向的不是同一个 scan， 这时候以 `SetState::currentScan` 为准。



ReplicaSetMonitor / SetState / Refresher / ScanState 具体关系参考下图：

![MongoDB-ReplicaSetMonitor.png](MongoDB-ReplicaSetMonitor.png)

 

## 探测目标 shard 要求

**只有以 ReplicaSet 方式启动的 shard(包含 config )，才会持有一个 `ReplicaSetMonitor` 类的成员。**即只有 ReplicaSet shard 才会被探测，以 Standalong 方式启动的 shard 不会进行探测

 

## 探测触发

ReplicaSetMonitor 中会**定期进行 shard 状态探测**，如果当前维系的状态**不能满足其他代码的 ReadPreference 要求，也会下发探测**

 

### 定期探测

`ReplicaSetMonitor` 初始化时会调用`_scheduleRefresh`，而后通过 `_scheduleRefresh` 和 `_doScheduledRefresh` 两个函数相互调用及 `TaskExecutor::scheduleWorkAt` 完成:

- `_scheduleRefresh`中通过使用 `TaskExecutor::scheduleWorkAt` 注册定时任务，定时任务负责调用 `_doScheduledRefresh`
- `_doScheduledRefresh` 负责调用 `Refresher::ensureScanInProgress` 进行 `Refresher` 新建与 探测下发。同时根据当前 `SetState` 的状态来决定下次刷新的周期，然后调用 `_scheduleRefresh` 注册新的定时任务

 

默认情况下，定时任务每隔 **kDefaultRefreshPeriod (30s)** 执行一次。但是如果 SetState 的 waiters 结构中非空，则会将周期调整为 **kExpeditedRefreshPeriod (500ms)** 。SetState 的 waiters 结构中 数据添加在下文按需探测时，数据清理在探测执行完成时。

 

### 按需探测

代码中需要按照某种 ReadPreference 获得 shard 的部分节点地址时，会调用 ReplicaSetMonitor 提供 `getHostsOrRefresh` 函数。如果当前持有的 shard 信息不能满足 ReadPreference 时，会构造一个 promise-future，返回 future ，然后将 promise 添加到 SetState 的 waiter 中，并立即下发刷新。

- 如果刷新结果能满足 ReadPreference ，则返回对应的host
- 如果不能满足，则会按照上文描述，下次 `_doScheduledRefresh` 被调用时，将刷新周期改为 **500ms**



```c++
void ReplicaSetMonitor::_doScheduledRefresh(const CallbackHandle& currentHandle) {
    ....
    Refresher::ensureScanInProgress(_state, lk);

    Milliseconds period = _state->refreshPeriod;
    if (_state->isExpedited) {
        if (_state->waiters.empty()) {
            // No current waiters so we can stop the expedited scanning.
            _state->isExpedited = false;
        } else {
            period = std::min(period, kExpeditedRefreshPeriod);
        }
    }

    _scheduleRefresh(_state->now() + period, lk);
}
```



 

## 探测方式

探测的核心 就是发送 **isMaster()** 命令并处理返回结果。

 

探测的中间状态记录在上文描述的 **ScanState** 中，主要持有几个 成员 来实现：

- **hostsToScan** : 待执行队列
- **possibleNodes** :     非primary节点返回的node集合
- **waitingFor** : 已经发送命令，等待返回的集合
- **triedHosts** : 已经下发isMaster命令的集合



探测过程有一个简单的状态机：**CONTACT_HOST**、**WAIT**、**DONE**，基于上面描述的 ScanState 持有的几个成员来判断：

- **CONTACT_HOST** : 链接当前实例。如果 **hostsToScan** 中有 host，则会返回该 host 并标记状态为 **CONTACT_HOST**
- **WAIT** : 等待 response 。如果 **hostsToScan** 为空，而 **waitingFor** 不为空则标记为 WAIT 状态
- **DONE** : Refresh 完成。如果 `_scan != _set->currentScan` 或者 **hostsToScan** & **waitingFor** 均为空，则为 **DONE** 状态

 

状态流转都在`getNextStep`中实现，用于`scheduleNetworkRequests`调度流转。

`IsMasterReply` : 记录isMaster必要的返回信息

 

### 请求下发

每一轮探测都会初始化一个新的 `ReplicaSetMonitor::Refresher`，随后在 `Refresher::startNewScan()` 中确定待探测的目标节点： `SetState` 的nodes中所有的节点。nodes中节点的确定方式：

* 初始化时，为 `seedNodes` 节点。addShard 的过程中会将shard的信息更新到 `config.shards` 集合中，随后各节点 ShardRegistry 定期（每隔30s）从 `config.shards` 中拉取shard信息。如果是未知的shard，则会创建 shardRemote，期间会创建对应的 `ReplicaSetMonitor`，而 `config.shards` 中的链接信息就是 `seedNodes`
* 初始化过后，nodes节点则为每次primary节点的isMaster命令返回的节点(hosts + passives)。

`scheduleNetworkRequests` 通过 `getNextStep` 依次从 **hostsToScan** 拿到一个host信息，并将 host 信息插入到 **waitingFor** 及 **triedHosts** 这 2 个 set 中。然后针对这个 host 调用 `scheduleIsMaster` 

 

`scheduleIsMaster` 构建一个 isMaster 的request ，提交给 executor ，并注册回调：

- 如果当前 Refresher 的 `_scan` 和 `_set->currentScan` 不同，则忽略返回结果
- 如果返回结果是 ok，则调用 `receivedIsMaster`；否则调用 `failedHost`

 

### response处理

`receivedIsMaster` : 将 host 从 waitingFor 中清理掉，然后构建一个 `IsMasterReply`

- 如果 `IsMasterReply` 的结果不是 ok ， 则会标记当前 host 为 fail
- 如果 reply 中 setName 和 _set 中的 name 不匹配，则会标记当前 host 为 fail
- 如果 reply 声明自己是 primary 节点， 则会调用 `receivedIsMasterFromMaster`
- 找到 primary 节点以后的 reply 处理： 将 reply 更新到对应的 `_set` 的 node 上，并在 notify 中判断是否有已经满足 `_set->waiter` 的 promise，有则返回
- 未找到 primary 节点前的 reply 处理： 将 reply 的 member 全部加到 **possibleNodes** 中，如果在 reply 中声明了 primary 节点的地址且该 primary 
- 在 **triedHosts** 中，则将该节点重新添加会 **hostsToScan**。然后将该 reply 记录到 **unconfirmedReplies** 中

 

`receivedIsMasterFromMaster` 处理逻辑比较多，简单整理了下：

- 有效性判断：判断 `configVersion` / `electionId` 的有效性
- 状态存储：将 reply 的结果更新到 `SetState` 中。更新nodes / seedNodes / seedConnStr / workingConnstr
  - nodes = reply中 hosts + passives，如果发生节点变化或者primary变化，则更新seedNodes = nodes
- 清理`ScanState` 中记录的信息：**triedHost** / **waiitingFor** / **unconfirmedReplies**
- 如果 primary 有变化，则会通知所有的 listener 状态变更。

 

`failedHost` : 将 host从 **waitingFor** 中移除，并将node标记为 fail

 

无论`receivedIsMaster` 还是 `failedHost` 执行完成后，都会调用 `scheduleNetworkRequests` 重新调度（可能有新加入 **hostsToScan** 的节点）

 

## 周边类

`ReplicaSetMonitorManager` 是负责 `ReplicaSetMonitor` 的管理类，维护一个 map 结构 （记录 setName 和 `ReplicaSetMonitor`），一个`TaskExecutor` 用于所有 `ReplicaSetMonitor` 命令执行，以及一个`ReplicaSetChangeNotifier`



## Questions

### Unable to reach primary for set

在进行 shard 整体迁移的时候，可能遇到该报错。原因是默认情况下 isMaster 探测的时间间隔是30s，而如果这30s之间，将shard的非hidden节点全部更换（下发isMaster的目标节点都不在当前replSet中），那么就无法探测到拓扑信息，进而导致 无法reach primary

```shell
#!/bin/bash

# reproduce step:
# 1. initial replSet has 3 members: IPADDR:8220 is primary, IPADDR:8221 is secondary, IPADDR:8222 is hidden
# 2. rs.remove("IPADDR:8221") manual
# 3. check mongos log : Updating ShardRegistry connection string for shard shard8220 from: shard8220/IPADDR:8220,IPADDR:8221 to: shard8220/IPADDR:8220
# 4. run this script immediately
mongo --host IPADDR --port 8220 --eval '
    var conf = rs.config();
    printjson(conf);
    conf.members[1].priority = 1;
    conf.members[1].hidden = false;
    printjson(conf);
    rs.reconfig(conf);
    rs.stepDown();
'

echo -e "\n\n\n"

mongo --host IPADDR --port 8222 --eval '
    rs.config();
    rs.remove("IPADDR:8220");
    rs.config();
'
```

