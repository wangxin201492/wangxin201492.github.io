---
title: MongoDB sharding aggregate实现原理
date: 2020-10-21 10:54:19
tags:
- MongoDB-OTT
categories:
- MongoDB
- sharding
- Aggregation
---



## 初始处理

mongos收到 `aggregate` 请求后，在确认完权限后，将请求转换为一个 `AggregationRequest` ，并随后交付 `ClusterAggregate::runAggregate()` 来完成。

* `AggregationRequest` : 这里只是将请求初步解析，主要是获取用户指定option中的配置，便于下一步处理，而`pipeline` 的各个 stage 则只会存储为一个BSON



> 进程初始化节点，会通过 `LiteParsedDocumentSource::registerParser()` & `DocumentSource::registerParser()` 来将 stage 里面的 关键字 与 具体的parser 关联到一个map结构中。

## ClusterAggregate::runAggregate()

* 首先，如果请求中有`fromMongos` / `needsMerge` / `mergeByPBRT` / `runtimeConstants` / `exchange`（这些参数都为内部参数，mongos中收到请求携带这些参数是不符合预期的），则uassert掉
* 将 `AggregationRequest` 中存储的 `pipeline` 进行实际解析(每个 stage 解析成一个 `LiteParsedDocumentSource`)，存储到 `LiteParsedPipeline` 维护的一个vector中
  * `LiteParsedPipeline` 和 上面说的 `AggregationRequest` 一样，也是进行了初步解析(semi-parsed version)。这里解析的目的是为了判断 各个stage 中存储的一些信息。比如 `getInvolvedNamespaces` / `startsWithCollStats` / `hasChangeStream` / `allowedToPassthroughFromMongos` / `allowShardedForeignCollection` / `assertSupportsReadConcern` / `verifyIsSupported` ，基本都是通过遍历上文提到的vector来砍断是否有某个stage符合 function 的要求的。
* 如果执行命令的namespace 是一个 CollectionlessAggregateNS (="$cmd.aggregate") 或者 pipeline中有 changeStream 的 stage，那么标记 该请求 `mustRunOnAll`
* 判断是否可以将请求发送给 primaryShard 处理：如果 `namespace和involvedNamespace都是unsharded的` && `mustRunOnAll == false` && `各个stage均允许PassthroughFromMongos`，那么**直接发送给 primaryShard 处理**
* 获取 namespace 的 collation & uuid 信息
* 构建一个 `ExpressionContext` ：主要还是把 `AggregationRequest` 中解析的参数传递过来
* `Pipeline::parse()` 来构建一个 `Pipeline`。
  * 这里首先将 `AggregationRequest` 中存储的 rawPipeline 中每个 stageObj 解析成 `DocumentSource`(通过上文提到的 `DocumentSource::registerParser()` 来解析)，存储在 `Pipeline` 的 `_sources` 中。
  * 然后对 `Pipeline` 进行 `validate()` && `stitch()`
    * validate: 每个 `DocumentSource` 都会设置自己的 constraints ，依次判断stage是否符合constraints
    * stitch: 将用户传入的 stage 串联起来
* `Pipeline::optimizePipeline()` 进行一些 stage 顺序上的优化。每个stage里面声明了自己可以做的优化方式.
* 如果 pipeline 必须在 mongos上执行，那么 **`runPipelineOnMongoS` 来完成剩余逻辑**
* 如果流程顺利完成，那么**进行 `sharded_agg_helpers::dispatchShardPipeline()` ，这里尝试将 pipeline 进行拆分并将请求发送给相关的shard。并获取结果：`shardDispatchResults`**
* 对于 `shardDispatchResults` 的处理：
  * 如果这是一个 explain，则直接将结果转换成 explain 并**返回**
  * 如果结果中 remoteCursor 的数量为1，即pipeline 未拆分，只发送给一个shard。那么存储 cursor，并将结果返回
  * 如果 标识有 `exchangeSpec` ，那么额外进行 `dispatchExchangeConsumerPipeline`
  * 最后将结果进行 merge：`dispatchMergingPipeline`



## sharded_agg_helpers::dispatchShardPipeline()

## dispatchMergingPipeline()
