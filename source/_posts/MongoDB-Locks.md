---
title: MongoDB Locks 实现简析
date: 2020-08-03 22:05:01
tags:
- MongoDB-OTT
categories:
- MongoDB
---



MongoDB代码内部还有锁的机制来避免并发冲突，MongoDB中有4种锁的类型，分别是：

* `MODE_IS` : Intent shared，意向读锁
* `MODE_IX` : Intent exclusive，意向写锁
* `MODE_S` : Shared，读锁
* `MODE_X` : Exclusive，写锁

下面是这4种锁的互斥矩阵：

```
/**
 * LockMode compatibility matrix.
 *
 * This matrix answers the question, "Is a lock request with mode 'Requested Mode' compatible with
 * an existing lock held in mode 'Granted Mode'?"
 *
 * | Requested Mode |                      Granted Mode                     |
 * |----------------|:------------:|:-------:|:--------:|:------:|:--------:|
 * |                |  MODE_NONE   | MODE_IS |  MODE_IX | MODE_S |  MODE_X  |
 * | MODE_IS        |      +       |    +    |     +    |    +   |          |
 * | MODE_IX        |      +       |    +    |     +    |        |          |
 * | MODE_S         |      +       |    +    |          |    +   |          |
 * | MODE_X         |      +       |         |          |        |          |
 */
```

对于同一个对象进行并发控制，具备**读(`S`)/写(`X`)锁**2个概念即可以解决。MongoDB 为了更好的管理不同层次之间锁的互斥关系，引入了**意向读(`IS`)/意向写(`IX`)锁**。

>  比如一个集合正在进行前台创建索引，是不可以进行其他写操作的，这是因为前台创建索引会给集合增加一个 X锁 ，而写操作在操作文档之前会先尝试给集合增加一个 IX锁，X锁 和 IX锁是不共存的，所以无法进行写入操作。



来看下获取 Collection 对象时的流程（代码中在 `AutoGetCollection` 中完成）：在获取 Collection 对象之前需要先获取一个 Database 对象，然后对 Collection 加指定锁。而在获取 Database 对象前会先加一个 Global级别的意向锁，随后对 Database 加锁。

> 也就是说，在对对象操作之前，需要按照 Collection -> Database -> Global 递归的向上加锁后，才会在下级进行加锁操作。



### 对 Global 加锁前，有2个依赖的 ticket 获取

如果 `FlowControlTicketholder`(4.2新增的流控机制) 开启，且当前为写操作，那么会首先从 `FlowControlTicketholder` 获取一个 ticket（该 ticket 受同步延迟影响）

```c++
void LockerImpl::getFlowControlTicket(OperationContext* opCtx, LockMode lockMode) {
    auto ticketholder = FlowControlTicketholder::get(opCtx);
    if (ticketholder && lockMode == LockMode::MODE_IX && _clientState.load() == kInactive &&
        opCtx->shouldParticipateInFlowControl() && !_uninterruptibleLocksRequested) {
        _clientState.store(kQueuedWriter);
        ticketholder->getTicket(opCtx, &_flowControlStats);
    }
}
```



从 `TicketHolder` 拿到一个 ticket，这个 ticket 主要是存储引擎传递过来可以并发操作的数量，以 wiredTiger 为例，读写并发数均为 **128**。随后先标记为 queue 状态，再 `waitForTicket()` ，如果 ticket 充足，则即刻获取到锁（`waitForTicket()`立即完成），进入 active 状态，否则会一直在 `waitForTicket()` 中。

```c++
bool LockerImpl::_acquireTicket(OperationContext* opCtx, LockMode mode, Date_t deadline) {
    const bool reader = isSharedLockMode(mode);
    auto holder = shouldAcquireTicket() ? ticketHolders[mode] : nullptr;
    if (holder) {
        _clientState.store(reader ? kQueuedReader : kQueuedWriter);
        OperationContext* interruptible = _uninterruptibleLocksRequested ? nullptr : opCtx;
        if (deadline == Date_t::max()) {
            holder->waitForTicket(interruptible);
        } else if (!holder->waitForTicketUntil(interruptible, deadline)) {
            return false;
        }
        restoreStateOnErrorGuard.dismiss();
    }
    _clientState.store(reader ? kActiveReader : kActiveWriter);
    return true;
}
```



### 具体加锁流程

对 Global / Database / Collection 具体的加锁，则是通过 `LockerImpl::lockBegin()` / `LockerImpl::lockComplete()` 来完成的。

* `lockBegin()` 则会根据 ResourceId 及 lockMode 来判断是否有锁冲突，如果有锁冲突，那么会进入 waiting 的状态
* `lockComplete()` 则会一直等待 lock的状态，直到加锁成功。

`lockBegin()` & `lockComplete()` 过程中还会对 acquireCount、acquireWaitCount、timeAcquiringMicros计数。



代码逻辑就不贴了，借助如下的基本结构及定义可以大致理解一下加锁过程：

* 首先锁是要加在Resource上的，以 `ResourceId` 来标识，`ResourceId`分为2部分，`ResourceType` & namespace。可以看到`Global`、`PBWM`、`RSTL`的 Resource 都只有一个 `ResourceId`，所以 ns使用了1，而其他的`Database`、`Collection`则会根据ns的不同有不同的lock。
* 实际加锁的时候，也是根据 `LockConflictsTable` 中的冲突关系来进行校验，判断是否可以直接加锁成功

```c++
// src/mongo/db/concurrency/lock_state.cpp
// Hardcoded resource IDs.
const ResourceId resourceIdLocalDB = ResourceId(RESOURCE_DATABASE, StringData("local"));
const ResourceId resourceIdOplog = ResourceId(RESOURCE_COLLECTION, StringData("local.oplog.rs"));
const ResourceId resourceIdAdminDB = ResourceId(RESOURCE_DATABASE, StringData("admin"));
const ResourceId resourceIdGlobal = ResourceId(RESOURCE_GLOBAL, 1ULL);
const ResourceId resourceIdParallelBatchWriterMode = ResourceId(RESOURCE_PBWM, 1ULL);
const ResourceId resourceIdReplicationStateTransitionLock = ResourceId(RESOURCE_RSTL, 1ULL);

// src/mongo/db/concurrency/lock_manager_defs.h
enum ResourceType {
    RESOURCE_INVALID = 0,
    /** Parallel batch writer mode lock */
    RESOURCE_PBWM,
    /** Replication state transition lock. */
    RESOURCE_RSTL,
    /**  Used for global exclusive operations */
    RESOURCE_GLOBAL,
    /** Generic resources, used for multi-granularity locking, together with the above locks */
    RESOURCE_DATABASE,
    RESOURCE_COLLECTION,
    RESOURCE_METADATA,
    /** Resource type used for locking general resources not related to the storage hierarchy. */
    RESOURCE_MUTEX,
    /** Counts the rest. Always insert new resource types above this entry. */
    ResourceTypesCount
};

// src/mongo/db/concurrency/lock_manager.cpp
static const int LockConflictsTable[] = {
    // MODE_NONE
    0,
    // MODE_IS
    (1 << MODE_X),
    // MODE_IX
    (1 << MODE_S) | (1 << MODE_X),
    // MODE_S
    (1 << MODE_IX) | (1 << MODE_X),
    // MODE_X
    (1 << MODE_S) | (1 << MODE_X) | (1 << MODE_IS) | (1 << MODE_IX),
};
```



参考：

1. MongoDB serverStatus.globalLock 深入解析：https://mongoing.com/archives/4768
2. 浅析MongoDB中的意向锁：https://developer.aliyun.com/article/655101
3. https://docs.mongodb.com/manual/faq/concurrency/
