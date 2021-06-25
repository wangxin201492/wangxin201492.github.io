---
title: MongoDB sharding实例 各类型节点(mongos/shard/config)如何管理集群状态
date: 2020-07-17 16:36:01
tags:
---

> 本文结合MongoDB4.2代码，描述 sharding集群 的各个类型节点(mongos/shard/config)内部如何管理集群状态。

首先，从mongos启动过程入手，来follow下集群状态管理相关组件的初始化工作：

`main()` --> `mongo::(anonymous namespace)::mongoSMain()` --> `mongo::(anonymous namespace)::main()` --> `mongo::(anonymous namespace)::runMongosServer()` --> `mongo::(anonymous namespace)::initializeSharding()` --> `mongo::initializeGlobalShardingState()`

`mongo::initializeGlobalShardingState()` 则是主要负责集群状态相关组件的初始化工作的，这里面主要初始化完成了一个 `Grid` 对象。

```c++
/**
 * Takes in the connection string for reaching the config servers and initializes the global
 * ShardingCatalogClient, ShardingCatalogManager, ShardRegistry, and Grid objects.
 */
Status initializeGlobalShardingState(OperationContext* opCtx,
                                     const ConnectionString& configCS,
                                     StringData distLockProcessId,
                                     std::unique_ptr<ShardFactory> shardFactory,
                                     std::unique_ptr<CatalogCache> catalogCache,
                                     rpc::ShardingEgressMetadataHookBuilder hookBuilder,
                                     boost::optional<size_t> taskExecutorPoolSize);
```



```c++
/**
 * Contains the sharding context for a running server. Exists on both MongoD and MongoS.
 */
class Grid {
     /**
     * Called at startup time so the global sharding services can be set. This method must be called
     * once and once only for the lifetime of the service.
     *
     * NOTE: Unit-tests are allowed to call it more than once, provided they reset the object's
     *       state using clearForUnitTests.
     */
    void init(std::unique_ptr<ShardingCatalogClient> catalogClient,
              std::unique_ptr<CatalogCache> catalogCache,
              std::unique_ptr<ShardRegistry> shardRegistry,
              std::unique_ptr<ClusterCursorManager> cursorManager,
              std::unique_ptr<BalancerConfiguration> balancerConfig,
              std::unique_ptr<executor::TaskExecutorPool> executorPool,
              executor::NetworkInterface* network);
    
private:
    std::unique_ptr<ShardingCatalogClient> _catalogClient;
    std::unique_ptr<CatalogCache> _catalogCache;
    std::unique_ptr<ShardRegistry> _shardRegistry;
    std::unique_ptr<ClusterCursorManager> _cursorManager;
    std::unique_ptr<BalancerConfiguration> _balancerConfig;

    // Executor pool for scheduling work and remote commands to shards and config servers. Each
    // contained executor has a connection hook set on it for sending/receiving sharding metadata.
    std::unique_ptr<executor::TaskExecutorPool> _executorPool;

    // Network interface being used by the fixed executor in _executorPool.  Used for asking
    // questions about the network configuration, such as getting the current server's hostname.
    executor::NetworkInterface* _network{nullptr};
}
```



`Grid` 负责集群状态相关组件的管理，包括：

* `ShardingCatalogClient` : 与 configServer 交互的入口 
* `CatalogCache` :   缓存的路由信息管理
* `ShardRegistry` :  shard信息管理，定期与 `config.shards` 交互拉取最新的shard信息
* `ClusterCursorManager` :  cursor 管理
* `BalancerConfiguration` :  Balancer配置管理
* `executor::TaskExecutorPool` :  ExecutorPool管理
* `executor::NetworkInterface` : 



### config节点初始化

`main()` --> `mongo::(anonymous namespace)::mongoDbMain()` --> `mongo::(anonymous namespace)::initAndListen()` --> `mongo::(anonymous namespace)::_initAndListen()` --> `mongo::initializeGlobalShardingStateForMongoD()` --> `mongo::initializeGlobalShardingState()`

### Shard节点初始化

`main()` --> `mongo::(anonymous namespace)::mongoDbMain()` --> `mongo::(anonymous namespace)::initAndListen()` --> `mongo::(anonymous namespace)::_initAndListen()` --> `mongo::ShardingInitializationMongoD::initializeShardingAwarenessIfNeeded()` --> `mongo::ShardingInitializationMongoD::initializeFromShardIdentity()` --> `mongo::ShardingInitializationMongoD::initializeShardingEnvironmentOnShardServer()` --> `mongo::initializeGlobalShardingStateForMongoD()` --> `mongo::initializeGlobalShardingState()`

> `ShardingInitializationMongoD` 类初始化的时候 将 `_initFunc` 初始化为 `initializeShardingEnvironmentOnShardServer`

