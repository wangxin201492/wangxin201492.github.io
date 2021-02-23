---
title: MongoDB sharding shardCollection实现流程
date: 2020-08-13 15:46:07
tags:
- MongoDB-OTT
categories:
- MongoDB
- sharding
- shardCollection
---

## mongos : `ShardCollectionCmd::run()`

mongos 收到 `shardCollection` 后，将请求解析并封装成一个 `_configsvrShardCollection`(ConfigsvrShardCollectionRequest) 请求，发送给 config节点

## config : `ConfigSvrShardCollectionCommand::run()`

1. 解析 `_configsvrShardCollection` 请求
2. 给 namespace & database 均增加 "shardCollection" 的分布式锁
3. 确认 database 已经是 sharded 的
4. 如果 database == "config" ，确认 namespace == "config.system.sessions" 且其 count() == 0
5. 构造一个 `_shardsvrShardCollection`(ShardsvrShardCollection) 请求，发送给 database 的 **primaryShard**

## shard : `ShardsvrShardCollectionCommand::run()`

解析 `_shardsvrShardCollection` 请求



将请求注册到 `ActiveShardCollectionRegistry` 中，

1. **如果没有同 namespace 的 shardCollection 运行，则注册成功，继续进行 shard 操作**
2. 如果有同 namespace 的请求，请求相同，则通过 promise/future 方式等待请求完成直接获取结果。请求不同则报错（*ConflictingOperationInProgress*）

主要的shard逻辑在 `shardCollection` 完成。

### 确认 `config.collections` 中，对 namespace 的 shard 操作是否与当前 请求相同

1. 不同则报错 ：*AlreadyInitialized*
2. 相同则返回 `config.collections` 中的结果，进而直接将对应的 collection 返回给上游
3. **不存在，则继续进行 shard 操作**

### 确认 collection 的一些状态 ： `calculateTargetState`

1. `config.chunks` 中有 namespace 的一些记录，报错：*ManualInterventionRequired*。需要手动请求可能因为之前 shard 操作导致有参与的chunk记录
2. 判断 namespace 索引状况：`createCollectionOrValidateExisting`
   1. 所有的 unique index 必须以 shardKey 为 prefix。否则：*InvalidOptions*
   2. 非空 collection 必须有一个有效 index
   3. 如果 shardKey 指定的是 unique的，必须有一个完全相同的 unique index
   4. 没有有效的 index， 且 collection 不为空则fail：*InvalidOptions*
   5. 空 collection 则创建 需要的 index
3. **isEmpty** : collection 是否为 空
4. **tags** : `config.tags` 中是否有 namespace 相关的 tag
5. **uuid** : 是否有有效的 uuid 或者生成一个新的
6. **fromMapReduce** : 是否来自于 mapReduce 请求
7. **splitPoint** : 基于 tag / shardKey类型 / 是否指定 initialSplitPoints(mapReduce) / numInitialChunks 等参数确定
   1. mapReduce 场景下，指定了 initialSplitPoints ： splitPoint = initialSplitPoints
   2. tag 为空 && shardKey 为 hashed && collection 为空，则将 chunk 按照 numInitialChunks 均分（未指定则默认为 shard * 2）
8. **numContiguousChunksPerShard** ： 每个 shard 持续的 chunk段



在计算`calculateTargetState` 前，shardCollection会

```c++
// From this point onward the collection can only be read, not written to, so it is safe to
// construct the prerequisites and generate the target state.
CollectionCriticalSection critSec(opCtx, nss)
```

通过一个 `CollectionCriticalSection` ，来将 collection 设置为只读，而在计算完`calculateTargetState` 后，

```c++
// From this point onward, the collection can not be written to or read from.
critSec.enterCommitPhase();
```

继续将其设置为不可读写（直到shardCollection完成）。



`CollectionCriticalSection` 是

###  确定 `ShardingOptimizationType` 并进行具体的 chunk 分配

1. **SplitPointsProvided** : 提供了 splitPoint
   1. 按照 splitPoint 将 chunk 分配到 shard 上(`generateShardCollectionInitialChunks`)
2. **TagsProvidedWithEmptyCollection** : tags 不为空 且 collection 不为空
   1. 按照 tags 将 chunk 分配到 shard 上(`generateShardCollectionInitialZonedChunks`)
3. **TagsProvidedWithNonEmptyCollection** : tags 不为空 collection 为空
   1. 将 **$minKey ~ $maxKey** 均分配到 primaryShard 上
4. **EmptyCollection** : collection 为空
   1. 将 **$minKey ~ $maxKey** 均分配到 primaryShard 上
5. **None** : tags 为空 && splitPoint 为空 && collection 不为空
   1. 将 maxChunkSizeBytes / maxChunkObjects / keyPattern 构造一个 `splitVector` 请求 发送给 primaryShard，获得一个 splitPoint 。然后根据结果将 chunk 拆分，但是都分配到 primaryShard

随后如果`ShardingOptimizationType != None && !fromMapReduce` ，发送给其他相关shard发送一个  `_cloneCollectionOptionsFromPrimaryShard` 来触发其从primaryShard获取该 Collection的相关信息。具体实现则使用 `moveChunk` 时用到的 `MigrationDestinationManager::cloneCollectionIndexesAndOptions()` 来完成。



### 更新config元信息及其他shard路由信息

上一步中会获得一个 `ShardCollectionConfig` 实际上是 `std::vector<ChunkType> chunks`。随后在 `writeChunkDocumentsAndRefreshShards` 中处理`ShardCollectionConfig`：

1. **writeFirstChunksToConfig**：写入 `config.chunks` 集合
2. **updateShardingCatalogEntryForCollection**：更新 `config.collections` 集合
3. **refreshAllShards**：强制刷新本地路由，基于最新的路由信息 构造一个 `_flushRoutingTableCacheUpdates` 请求，通知所有相关 shard

