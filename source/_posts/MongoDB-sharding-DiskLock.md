---
title: MongoDB sharding中分布式锁机制
date: 2020-03-24 19:11:00
tags:
- MongoDB-OTT
categories:
- MongoDB
- sharding
- DiskLock
---

[TOC]



sharding实例在 createCollection/dropCollection 等场景下，为了规避并发问题引入了分布式锁机制。分布式锁的信息记录在 [`config.locks`](https://docs.mongodb.com/manual/reference/config-database/index.html#config.locks)  集合中，结合 [`config.lockpings`](https://docs.mongodb.com/manual/reference/config-database/index.html#config.lockpings) 中的信息来完成相关逻辑实现。

## 分布式锁原理

### 1. `config.lockpings` 和 `config.locks` 集合中存储的内容

#### config.lockpings

[`config.lockpings`](https://docs.mongodb.com/manual/reference/config-database/index.html#config.lockpings) 集合跟踪记录分片集群中所有活跃的组件。



如果一个 mongos 运行在 example.com:30000 ，那么 `config.lockpings` 关于这个mongos的记录是这个样子的

```json
{ 
	"_id" : "example.com:30000:1350047994:16807", // 记录组件的标识， mongos 和 shard 节点均需要定期和 config 节点保持心跳
	"ping" : ISODate("2020-07-12T18:32:54.892Z") // 组件定期与 `lockpings` 集合保持心跳，即更新 `ping` 字段
}
```

> `_id` 字段内部称为 `processID` 。`processID` 对于 config 节点固定为"ConfigServer"；对于 mongos / shard 节点，则是以":"分隔的四段信息分别为：hostname / port / timestamp / 随机int64值。该4项信息在进程启动时即已决定，进程存活期间不会被修改。

#### config.locks

[`config.locks`](https://docs.mongodb.com/manual/reference/config-database/index.html#config.locks) 集合存储了分布式锁信息。

```json
{
   "_id" : "test.myShardedCollection", // 锁的名称，下文简称 lockName。对 database 或者 namespace 的部分场景操作需要获取分布式锁，所以一般 database 或者 namespace
   "state" : 2, // 锁的状态。0 表示 UNLOCKED，2 表示 LOCKED，1 表示 LOCK_PREP（仅对老版本3个config节点，目前代码中已无相关逻辑）
   "process" : "ConfigServer", // 即 processID。与上文讲到的 config.lockpings 集合中 _id 字段的取值是一样的
   "ts" : ObjectId("5be0b9ede46e4f441a60d891"), // 锁ID，下文简称 lockID。每次尝试获取分布式锁时的 锁ID 都是独有的。
   "when" : ISODate("2020-07-12T21:52:00.846Z"), // 获取锁的时间
   "who" : "ConfigServer:Balancer", // 获取锁的角色。  以":"分隔，第一段与 process字段 相同， 第二段 进程获取锁的线程名称。 ConfigServer:Balancer 表示 config进程的Balancer线程
   "why" : "Migrating chunk(s) in collection test.myShardedCollection" // 获取锁的原因
}
```



> `_id` 下文简称 `lockName`, `process` 下文简称 `processID`，`ts` 下文简称 `lockID`



### 2. 与 `config.lockpings` 和 `config.locks` 的基本交互

>  与上述2个集合的基础交互（`DistLockCatalog` 提供了与上述2个集合的基础交互动作，而 `DistLockCatalogImpl` 则是接口的具体实现）

**与 `config.lockpings` 的交互有 3 种场景：**

* `replSetDistLockPinger` 线程每隔 30s 通过一个 `upsert: true` 的 `findAndModify` 请求更新ping字段
* 系统 shutdown 时，会构造一个 `update: {}` 的 `findAndModify` 请求清理掉对应的document
* 此外在尝试获取分布式锁时，会获取对应组件上次心跳的时间，基于此判断组件是否已经丢失心跳，进而判断是否需要抢占锁

与 `config.locks` 交互便是我们关心的分布式锁交互过程：

* 组件获取到分布式锁有2种方式：
  * `grabLock` : **没有期望的 `lockName` 记录**或者**有期望的 `lockName` 记录且 `state = 0(UNLOCKED)`** ，通过更新 `state = 2(LOCKED)`, 同时更新`processID`/`lockID`/`who`/`when`/`why`字段的方式获取锁
  * `overtakeLock` : 对于**期望 `lockName`记录，其 `state = 0(UNLOCKED)` 或者 `lockID = oldTS` **，通过更新 `state = 2(LOCKED)`、更新 `lockID = newTS`、更新`processID`/`who`/`when`/`why`字段的方式抢占锁

### 3. 分布式锁获取逻辑

> 主要是通过 `ReplSetDistLockManager::lockWithSessionID()` 来完成的

1. 在预期情况下，应不存在对应的锁或者锁的状态应该是 `UNLOCKED`。所以首先通过 `grabLock` 的方式来获取锁
2. 如通过 `grabLock` 获取失败，则说明可能存在锁竞争的情况。则通过锁的名称来获取目前 `config.locks` 集合中记录的对应锁的信息
3. 基于锁的信息来判断：如果 **当前记录的分布式锁已经超时** 或者 **其对应的 processID 为当前请求的processID**， 则通过 `overtakeLock` 的方式抢占锁
   1. 当前记录的分布式锁已经超时：`ReplSetDistLockManager` 内部维护一个 `_pingHistory` 用于协助判断 `config.locks` 中记录的锁是否超时。`_pingHistory` 中记录了 `processID`/`pingValue`/`config节点的serverTime`/`lockID`/`config节点的electionId`。
      1. 无法在 `config.lockpings` 集合中找到对应组件的心跳记录，则认为**锁未超时**
      2. 如果 `_pingHistory` 中不存在对应 `lockName` 的记录，则认为**锁未超时**，并将相关结果记录到 `_pingHistor `中
      3. 如果与 `_pingHistory` 中记录相比，**锁的持有者心跳正常(`pingValue`字段持续更新)** 或者 **锁的`lockID`发生变化** 或者 **config发生主节点变更(`electionId`发生变化)** 则认为**锁未超时**，同时将相关结果记录到 `_pingHistory` 便于下次对比
      4. 最后，如果上述情况都没有发生，而 `_pingHistory` 中记录的 `config节点的serverTime` 时间与当前时间超过 15min，则认为**锁已经超时**
   2. 锁的信息中对应的 `ts` 为当前请求的`ts`
4. 如上述操作均失败，则等待后重新执行上述操作。直到超出指定的等待时间（`waitFor`）则返回`LockBusy`



借助简要代码看下：

> 入参中：`name` 为锁的名称（即 `_id` 字段），`lockSessionID` 为获取锁唯一的锁ID（即 `ts` 字段），`waitFor` 为预期等待锁的时间。


```c++
StatusWith<DistLockHandle> ReplSetDistLockManager::lockWithSessionID(OperationContext* opCtx,
                                                                     StringData name,
                                                                     StringData whyMessage,
                                                                     const OID& lockSessionID,
                                                                     Milliseconds waitFor) {
    ...

    // Distributed lock acquisition works by tring to update the state of the lock to 'taken'. If
    // the lock is currently taken, we will back off and try the acquisition again, repeating this
    // until the lockTryInterval has been reached. If a network error occurs at each lock
    // acquisition attempt, the lock acquisition will be retried immediately.
    while (waitFor <= Milliseconds::zero() || Milliseconds(timer.millis()) < waitFor) {
        ...

        auto lockResult = _catalog->grabLock(
            opCtx, name, lockSessionID, who, _processID, Date_t::now(), whyMessage.toString());

        auto status = lockResult.getStatus();

        if (status.isOK()) {
            ...
            return lockSessionID;
        }

        // Get info from current lock and check if we can overtake it.
        auto getLockStatusResult = _catalog->getLockByName(opCtx, name);
        const auto& getLockStatus = getLockStatusResult.getStatus();
        ...

        // Note: Only attempt to overtake locks that actually exists. If lock was not
        // found, use the normal grab lock path to acquire it.
        if (getLockStatusResult.isOK()) {
            auto currentLock = getLockStatusResult.getValue();
            auto isLockExpiredResult = isLockExpired(opCtx, currentLock, lockExpiration);

            if (isLockExpiredResult.getValue() || (lockSessionID == currentLock.getLockID())) {
                auto overtakeResult = _catalog->overtakeLock(xxx);
                ...
            }
        }
        ...

        if (waitFor == Milliseconds::zero()) {
            break;
        }

        const Milliseconds timeRemaining = std::max(Milliseconds::zero(), waitFor - Milliseconds(timer.millis()));
        sleepFor(std::min(kLockRetryInterval, timeRemaining));
    }

    return {ErrorCodes::LockBusy, str::stream() << "timed out waiting for " << name};
}
```



### 4. 加锁的场景

* collectin操作：`createCollection` / `dropCollection` / `shardCollection` 会同时对 collection 的 **namespace** 和 **database** 加锁
* database操作：`movePrimary` / `enableSharding` / `createDatabase` / `dropDatabase` 时会对 **database** 加锁，`dropDatabase` 还会依次对 db 下所有的 **collection** 加锁(`dropCollection`)

* chunk操作：Migrating chunk(s) in collection / merging chunks / splitting chunk
* map-reduce操作：mr-post-process



### 5. 解锁场景

* 一般情况在需要获取分布式锁的场景下，获取分布式锁成功会获得到一个 `DistLockManager::ScopedDistLock` 的对象，并在锁使用完成后触发该对象的析构函数，释放锁（修改state=UNLOCKED）。
* 另外如果 `grabLock` 时，如果获取失败返回异常是由于 config节点 状态异常导致，那么也会进行 `unlock` 方便下次可以直接 `grabLock` 完成加锁。
* MigrationManager 一些场景触发 // TODO
* 如果**在上述任何场景触发的unlock失败** 或者 **一些操作导致锁的状态未知** 后，都会加入到 `_unlockList` 队列，在 `replSetDistLockPinger` 定期执行时也会重新进行 unlock 操作



---

## 分布式锁代码解析

### 1. 核心类说明

####  `DistLockCatalogImpl : DistLockCatalog` : 对分布式锁的一些具体操作

```
/**
 * Interface for the distributed lock operations.
 */
class DistLockCatalog {}
```

1. 对config.lockpings的基础操作：ping/getPing/stopPing
2. 获取分布式锁or config的信息：getServerInfo/getLockByTS/getLockByname
3. 对锁的操作(config.locks)：grabLock/overtakeLock/unlock/unlockAll

其中`grabLock` 和 `overtakeLock` 是两个核心的获取锁的方法：

1. `grabLock` : 将`lockID`的锁更新为指定的`lockSessionID`
2. `overtakeLock` : 强制将锁的持有者从`currentHolderTS`更改为`lockSessionID`

#### `ReplSetDistLockManager : DistLockManager` : 分布式锁的一些接口，主要封装DistLockCatalogImpl而实现

```
/**
 * Interface for handling distributed locks.
 *
 * Usage:
 *
 * auto scopedDistLock = mgr->lock(...);
 *
 * if (!scopedDistLock.isOK()) {
 *   // Did not get lock. scopedLockStatus destructor will not call unlock.
 * }
 *
 * // To check if lock is still owned:
 * auto status = scopedDistLock.getValue().checkStatus();
 *
 * if (!status.isOK()) {
 *   // Someone took over the lock! Unlock will still be called at destructor, but will
 *   // practically be a no-op since it doesn't own the lock anymore.
 * }
 */
class DistLockManager {}
```

1. 持有一个线程`replSetDistLockPinger`，用户定时与config.lockpings心跳，并对需要unlock的锁进行unlock
2. 提供对锁处理的一些方法：
   1. 加锁：lock/lockWithSessionID/tryLockWithLocalWriteConcern
      1. lock通过调用lockWithSessionID来实现
   2. 解锁：unlock/unlockAll
3. 持有一个内部类ScopedDistLock : 一个RAII风格的类，持有锁的基础信息

### 2. 初始化

![MongoDB-sharding-DiskLockManager](https://wangxin201492.github.io/techImages/MongoDB-sharding-DiskLockManager.png)

mongos初始化时会生成一个与host、port、时间戳、随机值有关的一个`distLockProcessId`作为`ReplSetDistLockManager`的唯一标识，并在makeCatalogClient中完成对DistLockCatalogImpl、ReplSetDistLockManager、ShardingCatalogClientImpl的初始化

* DistLockCatalogImpl : 是`DistLockCatalog`的具体实现。默认初始化方法存储了`config.locks`，`config.lockpings`表名
* ReplSetDistLockManager : 是`DistLockManager`的具体实现。初始化方法存储了上面提到的`distLockProcessId`, `DistLockCatalog`并完成了`pingInterval`, `lockExpiration`的初始化，其中`pingInterval`默认为30s，`lockExpiration`默认为15min
* ShardingCatalogClientImpl : 是`ShardingCatalogClient`的具体实现。初始化方法存储了上面提到的`DistLockManager`。*(该类只提供了一个获取DistLockManager的方式及start、shutdown的方法，与DistLockManager无其他关系)*

然后将ShardingCatalogClientImpl作为一个数据成员存储在全局的Grid中

### 3. replSetDistLockPinger线程

#### 线程启动

grid初始化完成后，紧接着会调用`grid->catalogClient()->startup();`，该语句实际上最终调用到`ReplSetDistLockManager::startUp()`，启动一个replSetDistLockPinger线程，线程的具体执行在`ReplSetDistLockManager::doTask()`中

#### 线程逻辑 : doTask

1. config.lockpings交互：调用DistLockCatalog::ping()，构造一个findAndModify请求根据processID更新ping字段(upsert=true)。并更新本地的elapsedSincelastPing，如果与上次ping时间超过_pingInterval*10 则打印warning日志
2. unlock：遍历本地的 `_unlockList` ，对需要unlock的锁调用DistLockCatalog::unlock()。如果返回失败则打印warning日志并重新加入_unlockList中
3. sleep _pingInterval 即15s

### 4. 触发分布式锁的场景

* chunk操作：Migrating chunk(s) in collection / merging chunks / splitting chunk
* db or collection操作：movePrimary/enableSharding/dropCollection ...
* map-reduce操作：mr-post-process

#### collection 操作

| whyMessage         | _id       | function | file                                                         |
| ------------------ | --------- | -------- | ------------------------------------------------------------ |
| "createCollection" | database  | lock     | src/mongo/db/s/config/configsvr_create_collection_command.cpp |
| "createCollection" | namespace | lock     | src/mongo/db/s/config/configsvr_create_collection_command.cpp |
| "dropCollection"   | database  | lock     | src/mongo/db/s/config/configsvr_drop_collection_command.cpp  |
| "dropCollection"   | namespace | lock     | src/mongo/db/s/config/configsvr_drop_collection_command.cpp  |
| "shardCollection"  | database  | lock     | src/mongo/db/s/config/configsvr_shard_collection_command.cpp |
| "shardCollection"  | namespace | lock     | src/mongo/db/s/config/configsvr_shard_collection_command.cpp |

#### database 操作

| whyMessage       | _id       | function | file                                                        |
| ---------------- | --------- | -------- | ----------------------------------------------------------- |
| "movePrimary"    | database  | lock     | src/mongo/db/s/config/configsvr_move_primary_command.cpp    |
| "enableSharding" | database  | lock     | src/mongo/db/s/config/configsvr_enable_sharding_command.cpp |
| "createDatabase" | database  | lock     | src/mongo/db/s/config/configsvr_create_database_command.cpp |
| "dropDatabase"   | database  | lock     | src/mongo/db/s/config/configsvr_drop_database_command.cpp   |
| "dropCollection" | namespace | lock     | src/mongo/db/s/config/configsvr_drop_database_command.cpp   |

#### chunk 操作

| whyMessage                                                   | _id       | function                     | file                                          |
| ------------------------------------------------------------ | --------- | ---------------------------- | --------------------------------------------- |
| "splitting chunk " << chunkRange.toString() << " in " << nss.toString() | namespace | lock                         | src/mongo/db/s/split_chunk.cpp                |
| "merging chunks in " << nss.ns() << " from " << minKey << " to " << maxKey | namespace | lock                         | src/mongo/db/s/merge_chunks_command.cpp       |
| "Migrating chunk(s) in collection " << migrateType.getNss().ns()) | namespace | tryLockWithLocalWriteConcern | src/mongo/db/s/balancer/migration_manager.cpp |
| "Migrating chunk(s) in collection " << nss.ns()              | namespace | lockWithSessionID            | src/mongo/db/s/balancer/migration_manager.cpp |

#### map-reduce操作

| whyMessage        | _id       | function | file                                            |
| ----------------- | --------- | -------- | ----------------------------------------------- |
| "mr-post-process" | namespace | lock     | src/mongo/s/commands/cluster_map_reduce_cmd.cpp |



