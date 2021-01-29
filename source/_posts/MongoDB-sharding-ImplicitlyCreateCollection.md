---
title: MongoDB sharding实例 隐式创建collection
date: 2020-07-31 13:21:15
tags:
- MongoDB-OTT
categories:
- MongoDB
- sharding
- ImplicitlyCreateCollection
---

## 1. mongos节点 收到用户 insert 请求

mongos 收到一个 insert 请求后，会经过如下流程处理将请求发送给 shard 节点：

```
ServiceEntryPointMongos::handleRequest() --> Strategy::clientCommand() --> runCommand() --> execCommandClient() --> ClusterWriteCmd::InvocationBase::run() --> ClusterWriteCmd::InvocationBase::runImpl() --> ClusterWriter::write() --> BatchWriteExec::executeBatch() --> MultiStatementTransactionRequestsSender()
```

在发送给 shard 节点的请求中，会通过 `allowImplicitCollectionCreation` 字段来控制是否允许 shard 节点隐式创建collection。sharding状态下，该字段基本为 false，即不允许 shard 节点隐式创建collection

* 对于 `insert` / `update` / `delete` 这三类请求，会在 `ClusterWriteCmd::InvocationBase::runImpl()` 中进行 `batchedRequest.setAllowImplicitCreate(false);` 
* `src/mongo/s/cluster_commands_helpers.cpp` 中也提供了 `scatterGatherVersionedTargetByRoutingTable` / `scatterGatherUnversionedTargetAllShards` / `scatterGatherOnlyVersionIfUnsharded` / `executeCommandAgainstDatabasePrimary` 函数。如果请求包中不显式声明 `allowImplicitCollectionCreation` ，则默认为 `false` 
  * 但是 **renameCollection** 及 **covertToCapped** 会将 `appendAllowImplicitCreate` 声明为 `true`(可以隐式创建collection)

> shard节点可以直接创建collection，即为**隐式创建collection**(rs实例均采用这种方式)。否则需要和 config 交互来完成 collection 的创建，即为**显式创建collection**



## 2. shard节点 收到 mongos节点 转发过来携带 `allowImplicitCollectionCreation=false`的 insert 请求

正常情况下， shard节点 按照如下流程处理 insert 请求：

```
ServiceEntryPointMongod::handleRequest() --> ServiceEntryPointCommon::handleRequest() --> receivedCommands() --> execCommandDatabase() --> runCommandImpl() --> WriteCommand::InvocationBase::run() --> CmdInsert::Invocation::runImpl() --> performInserts() --> insertBatchAndHandleErrors() --> acquireCollection() --> makeCollection() --> DatabaseImpl::userCreateNS() --> createCollection()
```



而由于 `allowImplicitCollectionCreation=false` ，在 `execCommandDatabase()` 时会将 `allowImplicitCollectionCreation` 设置到 `OperationShardingState` 中。

```c++
oss.setAllowImplicitCollectionCreation(allowImplicitCollectionCreationField);
```



`createCollection()` 中则获取该值，判断为false，进行uassert，产生异常，向上抛出。

```c++
uassert(CannotImplicitlyCreateCollectionInfo(nss), "request doesn't allow collection to be created implicitly", OperationShardingState::get(opCtx).allowImplicitCollectionCreation());
```



异常在 `insertBatchAndHandleErrors` 中被 catch 住，在 `handleError()` 中判断异常为 `CannotImplicitlyCreateCollectionInfo` ，将异常信息设置到 `OperationShardingState` 中。

```c++
if (ex.extraInfo<CannotImplicitlyCreateCollectionInfo>()) {
  auto& oss = OperationShardingState::get(opCtx);
  oss.setShardingOperationFailedStatus(ex.toStatus());
  ...
}
```



至此原有处理流程结束， 退回到 `execCommandDatabase()` 中触发析构逻辑，在 `ScopedOperationCompletionShardingActions` 进行析构的时候处理之前设置到 `OperationShardingState` 中的异常状态。

```c++
auto scoped = behaviors.scopedOperationCompletionShardingActions(opCtx);
```



`ScopedOperationCompletionShardingActions` 的析构函数中，会通过 `onCannotImplicitlyCreateCollection` 函数向 config节点 下发一个 `_configsvrCreateCollection` 请求。发生异常则日志记录

```c++
ScopedOperationCompletionShardingActions::~ScopedOperationCompletionShardingActions() noexcept {
    if (auto cannotImplicitCreateCollInfo = status->extraInfo<CannotImplicitlyCreateCollectionInfo>()) {
        if (ShardingState::get(_opCtx)->enabled()) {
            auto handleCannotImplicitCreateStatus = onCannotImplicitlyCreateCollection(_opCtx, cannotImplicitCreateCollInfo->getNss());
            if (!handleCannotImplicitCreateStatus.isOK())
                log() << "Failed to handle CannotImplicitlyCreateCollection exception" << causedBy(redact(handleCannotImplicitCreateStatus));
        }
    }
}
```



## 3. config节点 处理 `_configsvrCreateCollection` 请求

config节点 收到 `_configsvrCreateCollection` 请求后，在 `ConfigSvrCreateCollectionCommand::typedRun()` 中进行处理

1. 首先通过和 `config.locks` 交互获取 `collection` 对应 namespace 和 database 的分布式锁
2. 然后通过 `ShardingCatalogManager::createCollection()` 和 primaryShard 交互，完成collection创建



## 4. shard节点 处理 `_configsvrCreateCollection` 结果

如 **阶段2** 中提到 `ScopedOperationCompletionShardingActions` 代码中，如 `_configsvrCreateCollection` 结果未按预期完成，则打印"Failed to handle CannotImplicitlyCreateCollection exception"日志记录。随后继续进行 `insert` 请求处理，此时因发生了 `CannotImplicitlyCreateCollectionInfo` 异常，所以最终 shard节点 向 mongos节点 返回一个 "request doesn't allow collection to be created implicitly" 错误：

```json
{ 
  n: 0,
  writeErrors: [ { 
    index: 0,
    code: 227,
    ns: "xxx.xxx",
    errmsg: "request doesn't allow collection to be created implicitly" 
    } ],
  opTime: { ts: Timestamp(1596026022, 1), t: 87 },
  electionId: ObjectId('7fffffff0000000000000057'),
  ok: 1.0
}
```



## 5. mongos节点 处理第一次 `insert` 结果

如 **阶段1** 中描述，mongos节点 `BatchWriteExec::executeBatch()` 中构造 `MultiStatementTransactionRequestsSender()` 向下游分发，并等待返回结果。

收到结果后，通过 `BatchWriteOp::noteBatchResponse()` 处理 response

```c++
// Dispatch was ok, note response
batchOp.noteBatchResponse(*batch, batchedCommandResponse, &trackedErrors);
```



`BatchWriteOp::noteBatchResponse()` 的入参中有一个 `BatchedCommandResponse` 对象记录所有节点返回的结果。而 `BatchWriteOp` 中的 `_writeOps` 则记录了向各个节点下发的请求，根据response结果一一标记对应的request。最终在 `WriteOp::_updateOpState()` 中对 当前writeOp 进行标记。如果 `writeError == StaleShardVersion` 或者 `writeError == CannotImplicitlyCreateCollection`，则当前 writeOp 的状态仍然保留为 `WriteOpState_Ready`，而如果遇到其他异常则被标记位 `WriteOpState_Error`，请求正常完成则被标记为 `WriteOpState_Completed`。

```c++
// Finish the response (with error, if needed)
if (NULL == writeError) {
    if (!ordered || !lastError) {
        writeOp.noteWriteComplete(*write);
    } else {
        // We didn't actually apply this write - cancel so we can retarget
        dassert(writeOp.getNumTargeted() == 1u);
        writeOp.cancelWrites(lastError);
    }
} else {
    writeOp.noteWriteError(*write, *writeError);
    lastError = writeError;
}
```



随后在 `BatchWriteExec::executeBatch()` 中判断 `batchOp.isFinished()` (有 `WriteOpState_Error` 则为 true，有`WriteOpState_Ready` 则为false)，此时由于 **阶段4** 中返回的是 `CannotImplicitlyCreateCollection`，所以 `batchOp.isFinished()` 的结果为 false。则继续构建 insert 请求下发给 shard节点进行重试。



## 6. shard节点 处理第二次 `insert` 请求 & mongos节点 处理第二次 `insert` 结果

由于 **阶段3** 中 config节点 已经触发 shard节点 完成collection的创建了，所以此时按照 **阶段2** 中的正常流程 `insertBatchAndHandleErrors` 继续处理完成，最终返回用户一个处理完成的 insert 请求结果

```json
{ 
  n: 1,
  opTime: { ts: Timestamp(1596026028, 6), t: 87 },
  electionId: ObjectId('7fffffff0000000000000057'),
  ok: 1.0
}
```



mongos节点 收到结果后按照 **阶段5** 中描述的由于没有error，则将结果组装后正常返回给上游用户

