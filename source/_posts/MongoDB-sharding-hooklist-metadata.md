---
title:mongos hooklist&metadata的故事
date: 2020-04-03 10:59:50
tags:
- MongoDB-OTT
categories:
- MongoDB
- sharding
- metadata
---



```c
static constexpr std::array<SpecialArgRecord, 26> specials{{
    //                                       /-isGeneric
    //                                       |  /-stripFromRequest
    //                                       |  |  /-stripFromReply
    {"$audit"_sd,                            1, 1, 0},
    {"$client"_sd,                           1, 1, 0},
    {"$configServerState"_sd,                1, 1, 1},
    {"$db"_sd,                               1, 1, 0},
    {"allowImplicitCollectionCreation"_sd,   1, 1, 0},
    {"$oplogQueryData"_sd,                   1, 1, 1},
    {"$queryOptions"_sd,                     1, 0, 0},
    {"$readPreference"_sd,                   1, 1, 0},
    {"$replData"_sd,                         1, 1, 1},
    {"$clusterTime"_sd,                      1, 1, 1},
    {"maxTimeMS"_sd,                         1, 0, 0},
    {"readConcern"_sd,                       1, 0, 0},
    {"databaseVersion"_sd,                   1, 1, 0},
    {"shardVersion"_sd,                      1, 1, 0},
    {"tracking_info"_sd,                     1, 1, 0},
    {"writeConcern"_sd,                      1, 0, 0},
    {"lsid"_sd,                              1, 0, 0},
    {"txnNumber"_sd,                         1, 0, 0},
    {"autocommit"_sd,                        1, 0, 0},
    {"coordinator"_sd,                       1, 0, 0},
    {"startTransaction"_sd,                  1, 0, 0},
    {"stmtId"_sd,                            1, 0, 0},
    {"$gleStats"_sd,                         0, 0, 1},
    {"operationTime"_sd,                     0, 0, 1},
    {"lastCommittedOpTime"_sd,               0, 0, 1},
    {"readOnly"_sd,                          0, 0, 1}}};
```



## metadata

|      fieldKey      |                    class                     | description                                                  |
| :----------------: | :------------------------------------------: | ------------------------------------------------------------ |
|      $client       | ClientMetadata / ClientMetadataIsMasterState | 从isMaster命令中解析出来的客户端信息("The ClientMetadata class is responsible for parsing the client metadata document that is received in isMaster from clients.")，主要包含 `application` , `driver` , `os` , `mongos`字段，其中当client是mongos时会有 `mongos` 字段（包含当前进程ip&port, mongos版本, client地址等信息），其余资源为client按照实际情况添加。 |
| $configServerState |             ConfigServerMetadata             | config server的metadata信息，存在于 shard 和 mongos 交互的 request和response 中。目前只包含 `opTime` 的信息。如果是config实例 `configsvr optime` 从 `ReplicationCoordinator` 中获取，而如果是shard或者mongos实例，则为 `Grid->configOpTime()` |
|       $audit       |         impersonated_user_metadata.h         | 已经授权的用户信息，只会出现在 mongos 向 shard 发送的request中，包含 `$impersonatedUsers` & `$impersonatedRoles` 。这里使用 `$audit` 是因为在4.2版本以前，企业版的审计系统已经使用该字段存储相关信息，这里复用该字段以便向后兼容 |
|    $clusterTime    |             LogicalTimeMetadata              | 逻辑时间，包含 `clusterTime` & `signature`                   |
|  $oplogQueryData   |              OplogQueryMetadata              |                                                              |
|     $replData      |               ReplSetMetadata                |                                                              |
|     $gleStats      |               ShardingMetadata               | getLastError信息，妥协response中的metadata. Shard将此信息添加到命令回复中，mongos 使用该信息来处理getLastError |
|   tracking_info    |               TrackingMetadata               | 跟踪命令执行数据信息，出现在通过 `ShardRemote` 操作的命令（主要是 `catalog` 操作）。不包括复制系统、查询请求和写入请求的命令 |



## hook

mongos的 `ThreadPoolTaskExecutor` (`: public TaskExecutor`)在 `NetworkInterfaceTL` (`: public NetworkInterface`)中进行具体的io操作，而初始化 `NetworkInterfaceTL` 时需要指定 `_onConnectHook`(`NetworkConnectionHook`) & `_metadataHook`(`rpc::EgressMetadataHook`)

### connectHook : `NetworkConnectionHook`

```
/**
 * An hooking interface for augmenting an implementation of NetworkInterface with domain-specific
 * host validation and post-connection logic.
 */
```



// TODO


### metadataHook : `rpc::EgressMetadataHook`

```
/**
 * An interface for augmenting egress networking components with domain-specific metadata handling
 * logic.
 *
 * TODO: At some point we will want the opposite of this interface (readRequestMetadata,
 * writeReplyMetadata) that we will use for ingress networking. This will allow us to move much
 * of the metadata handling logic out of Command::run.
 */
```

`EgressMetadataHook` 用于做扩容出口网络的组件，提供 `writeRequestMetadata` 将源信息写入request， `readReplyMetadata` 读取reply中的源信息. `EgressMetadataHook` 有4个具体的实现，其中 `EgressMetadataHookList` 提供一个结构用于存储多个 `EgressMetadataHook` 列表。

* `CommittedOpTimeMetadataHook` : 用于处理 `_lastCommittedOpTime` 的hook
  * writeRequestMetadata : 对request不做任何操作
  * readReplyMetadata : 对接收的reply，会提取当中的 `lastCommittedOpTime` 字段并通过 `Shard->updateLastCommittedOpTime` 更新 `_lastCommittedOpTime`
* `LogicalTimeMetadataHook` : 用于处理 `$clusterTime` 的hook
  * writeRequestMetadata : 通过 `LogicalTimeMetadata::writeToMetadata()` 将逻辑时钟的信息更新到 `$clusterTime`(包含 `clusterTime` & `signature`)
  * readReplyMetadata : 通过 `LogicalTimeMetadata::readFromMetadata` 获取 `$clusterTime` 中的信息，然后通过 `LogicalClock::advanceClusterTime` 进行逻辑时间更新
* `ShardingEgressMetadataHook` : sharding实例用于处理`configsvr optime` , `client metadata` , `auth metadata` 的hook。
  * writeRequestMetadata : 1. 通过 `writeAuthDataToImpersonatedUserMetadata` 将当前已经授权的用户信息(`auth metadata`)添加到 `$audit` 中；2. 通过 `ClientMetadataIsMasterState::writeToMetadata` 将客户端信息(`client metadata`)添加到 `$client` 中;  3. 通过 `ConfigServerMetadata::writeToMetadata` 将`configsvr optime`信息添加到 `$configServerState.opTime` 中
  * readReplyMetadata : 1. 通过 `_saveGLEStats` 保存GLE信息，`ShardingEgressMetadataHook`几个子类的实现都是空方法，但是注释中描述 mongod进程不做任何事情(no-op) , mongos实例会提取 reply中的 `$gleStats` 并填充到当前线程的client持有的 `ClusterLastErrorInfo`; // TODO 2. 通过 `_advanceConfigOpTimeFromShard` 更新 `configOpTime` 信息。如果是config实例，则从 `$replData` 源信息中提取，如果是shard或者mongos实例则从 `$configServerState.opTime` 中提取，最终 `grid->advanceConfigOpTime()` 完成设置。 

