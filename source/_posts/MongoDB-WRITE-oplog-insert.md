---
title: Oplog 是如何写入的？
date: 2021-06-04 10:09:10
tags:
- MongoDB-OTT
categories:
- MongoDB
- write
- oplog
---



[TOC]

## Oplog写入的相关组件

### OpObserver

```c++
/**
 * The OpObserver interface contains methods that get called on certain database events. It provides
 * a way for various server subsystems to be notified of other events throughout the server.
 *
 * In order to call any OpObserver method, you must be in a 'WriteUnitOfWork' ...
 */
class OpObserver {}
```

`OpObserver` 包含了各种事件对应的 methods（ `onInserts` / `onUpdate` / `onDelete` 等等事件），这样便于其他子系统知晓这些event。调用任何的 `OpObserver` 方法都需要在 `WriteUnitOfWork` 中，也就意味着所有需要的lock都已经被 `WriteUnitOfWork` 持有。



mongod进程初始化的时候会 add 多个 `OpObserver` 的实现：`OpObserverShardingImpl` / `AuthOpObserver` ，对于shardServer会add `ShardServerOpObserver` ，对于 configServer 会 add `ConfigServerOpObserver`：

* `AuthOpObserver` 只监听跟权限变更相关的oplog，具体实现在 `AuthzManagerExternalStateLocal::logOp()`
* `ShardServerOpObserver`  & `ConfigServerOpObserver` 都只会监听几个特定的oplog
* 主要的 oplog 写入行为在 `OpObserverShardingImpl`中完成，对于非事务的oplog，以 onInserts 为例，这里主要完成2件事情：
  * 通过 `_logOpWriter` 组装一条待写入 OplogDoc，通过 `_logOpsInner` 进行数据写入（此处又回调到`CollectionImpl::insertDocumentsForOplog`来完成实际的写入）
  * 旁路写入 `shardObserveInsertOp` ，如果当前shard是moveChunk的源，这里会从oplog中提取信息记录在内存中，方便增量同步用



### RecoveryUnit

```c++
/**
 * A RecoveryUnit is responsible for ensuring that data is persisted.
 * All on-disk information must be mutated through this interface.
 */
class RecoveryUnit {}
```

`RecoveryUnit` 负责确保数据的持久化，所有磁盘操作都必须通过该接口完成。实际上 `RecoveryUnit` 可以理解为 server层 的事务入口，storage层负责与引擎对接时需要实现这个接口。`RecoveryUnit` 提供了基础的 begin、commit、abort等操作，提供了Rollback&commit时的callback事件设置入口，同时也提供了时间戳设置的接口。



在 storageEngine init 的时候，会注册一个 `StorageClientObserver`，当opCtx初始化的时候，为opCtx设置当前使用引擎的 `RecoveryUnit`，以 wiredTiger为例，使用的就是 `WiredTigerRecoveryUnit`。



### WriteUnitOfWork

```c++
/**
 * The WriteUnitOfWork is an RAII type that begins a storage engine write unit of work on both the
 * Locker and the RecoveryUnit of the OperationContext. Any writes that occur during the lifetime of
 * this object will be committed when commit() is called, and rolled back (aborted) when the object
 * is destructed without a call to commit() or release().
 */
class WriteUnitOfWork {}
```

`WriteUnitOfWork` 是一个 RAII 风格类，操作 opCtx 的 `Locker` & `RecoveryUnit` 上开始存储引擎的写入工作单元。



## insert时 Oplog写入流程

insert 因为不涉及查询优化，这里的执行路径比较简单，先来看下insert时 oplog的插入流程。



在 `insertDocuments()` 时会开启一个 `WriteUnitOfWork`，并为请求分配optime，随后 `CollectionImpl::insertDocuments()` 来进行数据插入，其中会依次写数据、写索引，并将oplog传播给 `OpObserver`。

```c++
// src/mongo/db/ops/write_ops_exec.cpp:295
void insertDocuments( ... ) {
    // Intentionally not using writeConflictRetry. That is handled by the caller so it can react to
    // oversized batches.
    WriteUnitOfWork wuow(opCtx);
		...
    // 非事务的insert会分配时间戳
    auto oplogSlots = repl::getNextOpTimes(opCtx, batchSize);
  	...
    uassertStatusOK(
        collection->insertDocuments(opCtx, begin, end, &CurOp::get(opCtx)->debug(), fromMigrate));
    wuow.commit();
}

// src/mongo/db/catalog/collection_impl.cpp:432
Status CollectionImpl::insertDocuments( ... ) {
  	...
    const SnapshotId sid = opCtx->recoveryUnit()->getSnapshotId();

  	// 写数据、写索引
    status = _insertDocuments(opCtx, begin, end, opDebug);
    if (!status.isOK()) {
        return status;
    }
    invariant(sid == opCtx->recoveryUnit()->getSnapshotId());

  	// oplog 传播给 OpObserver
    getGlobalServiceContext()->getOpObserver()->onInserts(
        opCtx, ns(), uuid(), begin, end, fromMigrate);

  	// 注册 onCommit 回调
    opCtx->recoveryUnit()->onCommit(
        [this](boost::optional<Timestamp>) { notifyCappedWaitersIfNeeded(); });
		...
    return Status::OK();
}
```



注册的 `OpObserver` 会依次捕获该 oplog 写入事件进行处理，如上文提到，真正写 oplog 在 `OpObserverShardingImpl` 中完成，最终会构造一个 `DocWriter` 并调用 `CollectionImpl::insertDocumentsForOplog` 来进行实际写入。

