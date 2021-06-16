---
title: MongoDB sharding 节点路由信息管理（CatalogCache）
date: 2021-02-08 09:39:20
tags:
- MongoDB-OTT
categories:
- MongoDB
- sharding
- CatalogCache
---

[TOC]

## CatalogCache - 缓存已知路由信息

MongoDB sharding 环境中各节点(mongos/primary/secondary)在系统启动时，都会初始化一个 CatalogCache 用来缓存当前节点已知的路由信息。

CatalogCache通过 `_databases` & `_collectionsByDb` 来维护已知路由信息。

* `_databases` 是一个 **dbName->DatabaseInfoEntry** 的map结构。 `DatabaseInfoEntry`  中除了负责状态控制的 `needsRefresh` 以及 状态通知的 `refreshCompletionNotification` 外，主要的数据信息存储在 `DatabaseType` 结构中，这是一个和 `config.databases` 集合对应的数据结构
* 而 `_collectionsByDb` 则是一个 **dbName->(ns->CollectionRoutingInfoEntry)** 的两级map结构。`CollectionRoutingInfoEntry` 与 `DatabaseInfoEntry` 存储信息基本一致，除了 `needsRefresh` & `refreshCompletionNotification` 外，数据存储在 `RoutingTableHistory` 中，该结构与 `config.collections`集合+`config.chunks`集合 对应

CatalogCache中缓存的database&collection信息基本相似，主要来看下维护的collection路由信息。



### 路由信息获取 - `getCollectionRoutingInfo`

CatalogCache 提供 `getCollectionRoutingInfo` 来方便其他组件获取路由信息，比如mongos收到读写命令后都会先通过这个函数拿到路由信息以确定目标shard。

初始状态下（未分片状态），CatalogCache不存储任何的路由信息。所以从 `getCollectionRoutingInfo` 拿到的路由信息为空，则默认会发送给primaryShard



> 如果 primaryShard还没有确定，即 Database未创建。对于读请求，`ClusterFind::runQuery`时会返回空；对于写请求，首先会 `createShardDatabase`，这里实际是校验 db是否存在，如果不存在，则会给 configServer 发送 `_configsvrCreateDatabase` 请求，该请求处理时会确定primaryShard



### 路由信息交互 - `StaleConfigException`

mongos根据 CatalogCache 中存储的路由信息，将请求下发给后端shard，同时携带当前mongos已知的路由版本信息附加在请求的shardVersion字段中。后端shard节点收到请求后，会将 请求中的version信息(received) 和 本地的路由信息(wanted) 进行比对。如果版本不匹配（majorVersion 不同）则会返回给mongos一个 `StaleConfigException` 异常。

> 注意这里只要received与wanted的majorVersion版本不匹配，就会给mongos返回一个异常，mongos&shard都会去handle这个异常，版本较低的节点回去标记路由信息过期，随后mongos再次下发相同请求来进行真正的请求处理。



### 路由信息过期标记 - `needsRefresh`

CatalogCache 提供了 `invalidateShardedCollection` & `onStaleShardVersion` 2个函数来标记路由信息过期，标记的方式是标记`CollectionRoutingInfoEntry` 中的 `needsRefresh` = true。



### 路由刷新触发及并发控制 - `refreshCompletionNotification`

路由信息刷新仍然是在 CatalogCache 中 `getCollectionRoutingInfo` 完成的。同样是 `getCollectionRoutingInfo` ，除了上文说的初始状态（未分片状态）时拿到空路由信息外，都会拿到一个 `CollectionRoutingInfoEntry`。如果该entry被标记为needsRefresh，那么这里会通过加锁的机制来保证路由刷新触发且只被触发一次，其他的请求都会等待 `refreshCompletionNotification`。



### 路由信息刷新

CatalogCache 中除了维护上面说的db&collection的路由信息外，还维护一个CatalogCacheLoader，CatalogCacheLoader根据节点及属性不同会有不同的实现行为：

* mongos & configServer 使用`ConfigServerCatalogCacheLoader`
* shardServer 使用 `ShardServerCatalogCacheLoader` ，其中包装了一个`ConfigServerCatalogCacheLoader`
* readOnly状态的 shardServer 则使用 `ReadOnlyCatalogCacheLoader`。较少用到，不做描述

CatalogCache通过调用 CatalogCacheLoader的 getChunksSince函数 来获取路由信息，并在 refreshCollectionRoutingInfo 完成路由更新。

#### `ConfigServerCatalogCacheLoader`

`ConfigServerCatalogCacheLoader` 的 getChunksSince函数将请求调度到 **ConfigServerCatalogCacheLoader线程** 来完成。行为上，则是根据本地存储的 version信息中的epoch 与 config节点中config.collections中的epoch比较，如果相同则进行增量路由刷新，否则进行全量路由刷新。

> 上面提到的refreshCollectionRoutingInfo进行路由更新，也是在 **ConfigServerCatalogCacheLoader线程** 完成的

#### `ShardServerCatalogCacheLoader`

``ShardServerCatalogCacheLoader`本身会有一个 **ShardServerCatalogCacheLoader线程** 用于自身行为调度。

`ShardServerCatalogCacheLoader` 中包装了一个`ConfigServerCatalogCacheLoader`，这里等于是在 CatalogCache 和 `ConfigServerCatalogCacheLoader`中，插入了一层`ShardServerCatalogCacheLoader` 来完成shard节点（根据角色不同）一些特有的逻辑：

* primary仍然使用`ConfigServerCatalogCacheLoader`获取数据，随后持久化到 **"config.cache.chunks.xxx"** 集合中
  * primary节点的刷新行为在 `_schedulePrimaryGetChunksSince`中完成。这里进行增量刷新会基于 **"config.cache.chunks.xxx"** 中存储的最大版本来完成。而在拉取到数据后，会根据结果对该缓存集合进行drop 或者 update操作
* secondary 从 primary 同步数据，随后从 **"config.cache.chunks.xxx"** 中加载
  * secondary节点刷新行为在 `_runSecondaryGetChunksSince` 中完成。首先给 primary 节点下发一个 `forceRoutingTableRefresh` 请求，并获得响应时间戳 opTime，然后等待同步时间戳越过该opTime。然后从 **"config.cache.chunks.xxx"** 中加载数据



### CatalogCache状态查看

```javascript
> db.serverStatus().shardingStatistics.catalogCache
{
  "numDatabaseEntries" : NumberLong(<num>),
  "numCollectionEntries" : NumberLong(<num>),
  "countStaleConfigErrors" : NumberLong(<num>),
  "totalRefreshWaitTimeMicros" : NumberLong(<num>),
  "numActiveIncrementalRefreshes" : NumberLong(<num>),
  "countIncrementalRefreshesStarted" : NumberLong(<num>),
  "numActiveFullRefreshes" : NumberLong(<num>),
  "countFullRefreshesStarted" : NumberLong(<num>),
  "countFailedRefreshes" : NumberLong(<num>)
}
```

* numDatabaseEntries : catalog cache中 database entry的数量
* numCollectionEntries : catalog cache中 collection entry的数量
* **countStaleConfigErrors : 触发 stale config的次数**
* **totalRefreshWaitTimeMicros : 刷新metadata的等待总耗时**
* **numActiveIncrementalRefreshes : 目前等待增量catalog cache刷新完成的数量**
* **countIncrementalRefreshesStarted : 增量catalog cache开始刷新的次数**
* **numActiveFullRefreshes : 目前等待全量catalog cache刷新完成的数量**
* **countFullRefreshesStarted : 全量catalog cache开始刷新的次数**
* **countFailedRefreshes : 增量或全量catalog cache刷新失败的次数**
* **operationsBlockedByRefresh : [mongos 4.2.7] catalog cache刷新导致阻塞的operation总数**



---

## shard路由信息管理

除了CatalogCache，sharding环境下mongod对每个collection还会使用`CollectionShardingState`来维护额外的一些路由状态。

### 核心类关系

在 metadata_manager.cpp 注释中描述了该部分用到的几个核心类之间的关系：`CollectionShardingRuntime`(`CollectionShardingState`)  & `MetadataManager` & `CollectionMetadata` & `ScopedCollectionMetadata`。（该关系描述与目前4.2最新代码少有gap，但是不影响理解，下文依4.2版本代码进行关系描述）

```c++
// src/mongo/db/s/metadata_manager.cpp
//                                        ____________________________
//  (s): std::shared_ptr<>       Clients:| ScopedCollectionMetadata   |
//   _________________________        +----(s) manager   metadata (s)------------------+
//  | CollectionShardingState |       |  |____________________________|  |             |
//  |  _metadataManager (s)   |       +-------(s) manager  metadata (s)--------------+ |
//  |____________________|____|       |     |____________________________|   |       | |
//   ____________________v________    +------------(s) manager  metadata (s)-----+   | |
//  | MetadataManager             |   |         |____________________________|   |   | |
//  |                             |<--+                                          |   | |
//  |                             |        ___________________________  (1 use)  |   | |
//  | getActiveMetadata():    /---------->| CollectionMetadata        |<---------+   | |
//  |     back(): [(s),------/    |       |  _________________________|_             | |
//  |              (s),-------------------->| CollectionMetadata        | (0 uses)   | |
//  |  _metadata:  (s)]------\    |       | |  _________________________|_           | |
//  |                         \-------------->| CollectionMetadata        |          | |
//  |  _receivingChunks           |       | | |                           | (2 uses) | |
//  |  _rangesToClean:            |       | | |  _tracker:                |<---------+ |
//  |  _________________________  |       | | |  _______________________  |<-----------+
//  | | CollectionRangeDeleter  | |       | | | | Tracker               | |
//  | |                         | |       | | | |                       | |
//  | |  _orphans [range,notif, | |       | | | | usageCounter          | |
//  | |            range,notif, | |       | | | | orphans [range,notif, | |
//  | |                 ...   ] | |       | | | |          range,notif, | |
//  | |                         | |       | | | |              ...    ] | |
//  | |_________________________| |       |_| | |_______________________| |
//  |_____________________________|         | |  _chunksMap               |
//                                          |_|  _chunkVersion            |
//                                            |  ...                      |
//                                            |___________________________|
```



**summary**：`CollectionShardingState` / `CollectionShardingRuntime` 提供对外交互的接口函数；`ScopedCollectionMetadata` 为输出给外部接口的类；`MetadataManager` 负责核心逻辑处理；`CollectionMetadata` 负责信息存储。



#### `CollectionShardingState` / `CollectionShardingRuntime` - 对外接口

```c++
// src/mongo/db/s/collection_sharding_state.h
/**
 * Each collection on a mongod instance is dynamically assigned two pieces of information for the
 * duration of its lifetime:
 *  CollectionShardingState - this is a passive data-only state, which represents what is the
 * shard's knowledge of its the shard version and the set of chunks that it owns.
 *  CollectionShardingRuntime (missing from the embedded mongod) - this is the heavyweight machinery
 * which implements the sharding protocol functions and is what controls the data-only state.
 */
```

mongod实例上的每个集合在其生命周期内都会动态分配两条信息：`CollectionShardingRuntime` & `CollectionShardingState`，`CollectionShardingRuntime` 是 `CollectionShardingState` 的子类：

* `CollectionShardingState`主要提供2个功能：
  * 提供路由信息获取，以 `ScopedCollectionMetadata`向外输出。而这些路由信息的提供也是依靠 `CollectionShardingRuntime` 实现的_getMetadata() 来完成
  * 使用 `ShardingMigrationCriticalSection` 来完成对collection的一些读写控制

* 而 `CollectionShardingRuntime` 除了继承 `CollectionShardingState` 的功能以外，还实现了 sharding协议的一些功能，比如moveChunk相关、orphan文档相关等。而实现这些功能（包括上面提到的_getMetadata()）都是通过其内部封装的 `MetadataManager` 来完成的。

##### CollectionShardingStateFactory & CollectionShardingStateMap

 `CollectionShardingState` / `CollectionShardingRuntime` 的管理是由 `CollectionShardingStateMap` 来完成的，内部维护一个 **ns -> CollectionShardingState** 的map结构。而`CollectionShardingRuntime`的对象生成，则是由 `CollectionShardingStateFactoryShard` 负责，factory中有一个 taskExecutor 被所有 `CollectionShardingRuntime` 共享使用，其维护的线程名为 **CollectionRangeDeleter-TaskExecutor**。



#### `CollectionMetadata` - 信息存储

```c++
// src/mongo/db/s/collection_metadata.h
/**
 * The collection metadata has metadata information about a collection, in particular the
 * sharding information. It's main goal in life is to be capable of answering if a certain
 * document belongs to it or not. (In some scenarios such as chunk migration, a given
 * document is in a shard but cannot be accessed.)
 */
```

`CollectionMetadata`中有集合关于sharding的metadata，主要的功能是来回答这条文档是否属于当前shard。

`CollectionMetadataTracker` 负责维护 `CollectionMetadata` 的生命周期，其中的usageCounter标识其引用计数。



#### `MetadataManager` - 核心逻辑

每个 `CollectionShardingRuntime` 内部也会唯一对应一个 `MetadataManager`

`MetadataManager` 中的 `_metadata` 是一个list结构，存储 `CollectionMetadataTracker`。



#### `ScopedCollectionMetadata` - 输出类

提供一个 get() 函数来获取 `CollectionMetadata`，RAII风格类，构造&析构时对 `CollectionMetadataTracker`中的usageCounter进行增减。



#### `CollectionRangeDeleter` - 数据删除

`CollectionRangeDeleter` 负责存储待删除range及提供具体的删除逻辑。



待删除range信息存储在`Deletion`结构体中， 保存3个信息：range（`ChunkRange`） & when（`Date_t`）& notification，其中 when 为 **Date_t{}** 表示立即删除。

`CollectionRangeDeleter` 维护2个`std::list<Deletion>`用于存储待删除range：

*  `_orphans`：立即删除的range，即`Deletion` 中 when 值为 **Date_t{}** 
*  `_delayedOrphans`：指定时间删除的range，即`Deletion` 中 when 值不为 **Date_t{}** 



`CollectionRangeDeleter` 提供几个核心逻辑：

* `add` : 根据入参的 Deletion 信息，来将其存入 `_orphans` / `_delayedOrphans`。并根据当前队列状态，返回一个 Date_t 交由外部判断是否触发 delete
* `overlaps` :  // TODO
* `cleanUpNextRange` : 实际进行 range 删除，每次range 删除完成以后，会通过Deletion的notification notify其他线程。// TODO



### 功能实现

> 上文描述了核心类之间关系，接下来通过一个个应用场景来看下这几个类如何协同以及上文未提及的一些字段

#### shardVersion比对 - `StaleConfigException`

`CollectionShardingState`中提供了 checkShardVersionOrThrow() 来进行本地路由信息与请求中的shardVersion字段匹配的功能，如果不匹配，则会抛出一个 `StaleConfigException` 异常。

从 `OperationShardingState` 中获取请求中shardVersion字段信息：**receivedShardVersion**，从 `MetadataManager` 中获取最新的`CollectionMetadata`：**wantedShardVersion**，比较 **receivedShardVersion** & **wantedShardVersion** 是否匹配（majorVersion是否相等）
