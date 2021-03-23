---
title: MongoDB Sharding splitChunk实现流程
date: 2020-09-02 16:43:32
tags:
- MongoDB-OTT
categories:
- MongoDB
- sharding
- splitChunk
---

## 准备阶段

### autoSplit

> shard&config 节点，会有 PeriodicBalancerConfigRefresher 定时任务每隔 30s 进行 balancer&split 配置刷新
>
> mongos 节点，会有 Uptime-reporter 线程每隔 10s 进行 balancer&split 配置刷新

1. 每当有插入或者更新操作时，会触发 `ShardServerOpObserver` 的 `onInserts` / `onUpdate`
2. 这2个方法都会调用 `incrementChunkOnInsertOrUpdate` 来判断是否需要进行 split chunk：chunk size 超出配置值，且开启 autoSplit
3. 如果需要进行 split chunk， 则 `ChunkSplitter` 会接管后续的 split 流程
4. 通过 `splitVector` 来获取具体的 `splitPoint`，然后调用 `splitChunk` 进行实际的 split chunk 操作

#### 4.2 之前版本，autoSplit由mongos触发

mongos 中 `updateChunkWriteStatsAndSplitIfNeeded` 记录每个 chunk 写入的大小，如果总更新量超过 maxChunkSizeBytes / 5 ，则会向 chunk 所在的 shard 发送一个 `splitVector` 请求来询问是否需要进行 split 。如果需要，则向 该shard 再发送一个 `splitChunk` 请求 进行实际的 split

### split Command

1. mongos 收到 `split` 请求。判断合法性后计算 `splitPoint`，然后构造一个 `splitChunk` 请求，发送给 shard 节点
2. shard节点 收到 `splitChunk` 请求。解析相关参数，然后 调用 `splitChunk` 进行实际的 split chunk 操作



## splitChunk

1. shard 节点首先从 config 节点获取一个 **why = splitting chunk chunkRange in namespace** 的分布式锁
2. 然后发送一个 `_configsvrCommitChunkSplit` 请求给 config 节点
3. config 节点判断 config.chunks 中的内容与 请求参数 之间的合法性。
4. 然后构造一个 `applyOps` 请求进行 config.chunks 集合的更新，然后进行 config.changelog 集合
5. 上述流程结束后，会判断此次 split 的边界是否有 topChunk ，如果存在的话，则将该新的chunkRange返回上游。
   1. 如果是split command 的请求，不会对 topChunk 做任何处理
   2. 如果是 autoSplit ，会进行 topChunk 的处理，但是不是依赖这里返回的 topChunk，而是 ChunkSplitter 自行计算的结果
6. 析构，释放分布式锁。



## topChunk Optimization

包含 MinKey 或者 MaxKey 的 chunk 被称作 `topChunk`。如果 topChunk 被 split，那么可能用户有更大的概率对 shardKey 进行单调递增或单调递减的插入。这种情况下 `ChunkSplitter` 会主动向 configServer 请求尝试 moveChunk