---
title: MongoDB sharding实例 路由刷新策略
date: 2020-09-21 10:05:48
tags:
- MongoDB-OTT
categories:
- MongoDB
- sharding
- StaleConfig
---

## 1. mongos节点 收到用户 insert 请求

mongos 收到一个 insert 请求后，会经过如下流程处理将请求发送给 shard 节点：

```
ServiceEntryPointMongos::handleRequest() --> Strategy::clientCommand() --> runCommand() --> execCommandClient() --> ClusterWriteCmd::InvocationBase::run() --> ClusterWriteCmd::InvocationBase::runImpl() --> ClusterWriter::write() --> BatchWriteExec::executeBatch() --> MultiStatementTransactionRequestsSender()
```

在 `ClusterWriter::write()` 中会准备一个 `ChunkManagerTargeter` 来维保存目标 namespace 在 `CatalogCache` 中的路由信息，方便后续请求处理。

主要代码逻辑在 `BatchWriteExec::executeBatch()` 中，主要是封装 `BatchWriteOp` 的几个核心方法来完成具体的请求处理工作：

首先，根据请求的目标shard、order 等参数来将请求分成多个批次。

```c++
  Status targetStatus = batchOp.targetBatch(targeter, recordTargetErrors, &childBatches);
```

然后逐批构造请求，并将请求通过 `MultiStatementTransactionRequestsSender` 依次发送到shard，并等待接收结果。

```c++
  const auto request = [&] { const auto shardBatchRequest(batchOp.buildBatchRequest(*nextBatch));
  ...
  MultiStatementTransactionRequestsSender ars(
    opCtx,
    Grid::get(opCtx)->getExecutorPool()->getArbitraryExecutor(),
    clientRequest.getNS().db().toString(),
    requests,
    kPrimaryOnlyReadPreference,
    isRetryableWrite ? Shard::RetryPolicy::kIdempotent : Shard::RetryPolicy::kNoRetry);
```



在请求构造(`BatchWriteOp::buildBatchRequest()`)的的时候，设置请求的 `shardVersion`。这里实际是为请求设置了一个 `shardVersion` 字段，设置的值则为刚开始提到的 `CatalogCache` 中保存的 namespace 路由信息。

```c++
request.setShardVersion(targetedBatch.getEndpoint().shardVersion);
```



```json
{ 
  insert: "sCollName", 
  documents: [ { _id: ObjectId('5f685824c800cd1689ca3be8'), name: xxxx } ], 
  shardVersion: [ Timestamp(5, 1), ObjectId('5f3ce659e6957ccdd6a56364') ], 
  $db: "sdb"
}
```



## 2. shard节点 收到 mongos节点 转发过来携带 `shardVersion` 的 insert 请求

正常情况下， shard节点 按照如下流程处理 insert 请求：

```
ServiceEntryPointMongod::handleRequest() --> ServiceEntryPointCommon::handleRequest() --> receivedCommands() --> execCommandDatabase() --> runCommandImpl() --> WriteCommand::InvocationBase::run() --> CmdInsert::Invocation::runImpl() --> performInserts() --> insertBatchAndHandleErrors() --> acquireCollection()
```



在 `execCommandDatabase()` 中，会提取请求中的 `shardVersion`和`databaseVersion` 字段信息，存储到 `OperationShardingState` 中。

```c++
if (!opCtx->getClient()->isInDirectClient() &&
            readConcernArgs.getLevel() != repl::ReadConcernLevel::kAvailableReadConcern &&
            (iAmPrimary ||
             (readConcernArgs.hasLevel() || readConcernArgs.getArgsAfterClusterTime()))) {
  oss.initializeClientRoutingVersions(invocation->ns(), request.body);
}
```



只有执行了`oss.initializeClientRoutingVersions()`，后续才会进行shardVersion比对。那么不进行shardVersion比对的场景是：

1. reacConcern指定为available
2. 从secondary读取且readConcern没有指定任何的参数

### 2.1 是否可以进行写入

在 `acquireCollection()` 时，会判断请求是否可以进行写入( `assertCanWrite_inlock()` )。判断的标准则是比对 **当前节点存储的shardVersion** 和 **收到mongos发送来的shardVersion**，比较二者 `epoch & majorVersion` 是否相同，相同则认为可以进行写入。

```c++
// Can we write to this data and not have a problem?
bool isWriteCompatibleWith(const ChunkVersion& other) const {
  return epoch() == other.epoch() && majorVersion() == other.majorVersion();
}
```

否则会抛出一个 `StaleConfigInfo` 异常。

### 2.2 异常处理

该异常在 `insertBatchAndHandleErrors()` 中进行捕获，并 `handleError()`。对于异常为 `StaleConfigInfo` ，处理手段是将其设置到 `OperationShardingState` 中，并将其结果添加到向上游返回的输出中，且认为代码无须向下执行，继续直接返回给上游。

```c++
if (ex.extraInfo<StaleConfigInfo>()) {
        if (!opCtx->getClient()->isInDirectClient()) {
            auto& oss = OperationShardingState::get(opCtx);
            oss.setShardingOperationFailedStatus(ex.toStatus());
        }

        // Don't try doing more ops since they will fail with the same error.
        // Command reply serializer will handle repeating this error if needed.
        out->results.emplace_back(ex.toStatus());
        return false;
    }
```



在 `execCommandDatabase()` 进入析构流程时，析构 `ScopedOperationCompletionShardingActions` 会处理上面提及的 `StaleConfigInfo`。在 `onShardVersionMismatch` 接管异常处理工作，比对 本地shardVersion 是否大于 接收的版本。如果是则无需处理，否则进行 `forceShardFilteringMetadataRefresh` 强制刷新。



而无论是否进行了刷新，都会将 `StaleConfigInfo` 返回给上游 mongos



## 3. mongos 处理 `StaleConfigInfo`

续 **阶段1** ， `MultiStatementTransactionRequestsSender()` 收到结果以后，通过 `BatchWriteOp::noteBatchResponse()` 处理 response

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



随后仍然会将 `StaleConfigInfo` 异常信息添加到 targeter 中，并在接下来的逻辑中进行刷新（如果本地版本更老）

```c++
  if (!staleErrors.empty()) {
    noteStaleResponses(staleErrors, &targeter);
  }
  ...
  Status refreshStatus = targeter.refreshIfNeeded(opCtx, &targeterChanged);
```



刷新的方式是首先标记版本过期，需要刷新。然后重新 `init` 时会获取路由信息而出发刷新。

```c++
Status ChunkManagerTargeter::_refreshNow(OperationContext* opCtx) {
    Grid::get(opCtx)->catalogCache()->onStaleShardVersion(std::move(*_routingInfo));

    return init(opCtx);
}
```

