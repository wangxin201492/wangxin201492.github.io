---
title: MongoDB-sharding-JIRAs
date: 2021-06-03 11:04:20
tags:
- MongoDB-OTT
categories:
- MongoDB
- sharding
---



### Read from secondary got empty response

从secondary读取受路由刷新问题影响（defaultReadConcern：available & logic partition）获取空结果

> 2021.02.07 20:52

mongosA(路由为最新的) & mongosB & shardA(primaryShard) & shardB

稳定复现：
step1：mongosA上进行1. shardCollection({id: "hashed"}); 2. 插入数据（这里假设id:2的数据在shardB上）
step2：mongosB上find({id:2}).readPref("secondary")，查询不到数据。无论是否指定readConcern

原因：mongosB上路由不准确，且在请求的过程中没有进行路由刷新（StaleConfigInfo）

1. mongosB之前没有收到任何关于该collection的请求，所以认为其是 UNSHARDED，即chunkVersion=(0,0)
2. mongosB收到step2中的find请求后，转发给shardA(primaryShard)的secondary节点。如果这里不指定readConcern，那么默认是available（这种场景不进行路由比对)
3. 即使指定了readConcern，这里shardA的secondary节点会进行路由比对。那么这里received=(0,0),而本地路由wanted也是(0,0)，即secondary仍然认为路由是对齐的，响应请求。

> 本地路由wanted是(0,0)是因为路由信息同步过来但没有加载导致

目前secondary的路由信息是从primary的 config.cache.chunks.xxx 集合中（通过oplog）同步过来的，而数据同步过来后并没有加载到secondary的本地CatalogCache中。 

>  路由信息加载是lazy的（https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/README.md#when-the-routing-table-cache-will-refresh ），主要还是依靠请求交互，或者有类似dropCollection、moveChunk这样的场景来标记

而是等待该集合路由信息被标记为needRefresh才会走CatalogCacheLoader::getChunksSince()来触发路由刷新，这里secondary的行为是强制primary路由刷新，然后等待同步时间戳推进，此时会将路由信息加载到CatalogCache。

> Q：什么情况下集合路由信息会被标记为 needRefresh 呢？
>
> “主要还是依靠请求交互，或者有类似dropCollection、moveChunk这样的场景来标记”。具体场景我也没有完整梳理，对应的就是CatalogCache::CollectionRoutingInfoEntry中的needsRefresh字段，基本上获取路由信息的时候都会check这个字段的

解决方法目前看除了手动刷新外，在用户请求可以访问所有的mongos&secondary情况下是基本不存在问题的。
像紫龙的问题，是插入数据后只读取其中一个数据，即mongosA进行数据插入，且插入的数据全部都到了shardB，而mongoB由于无路由信息，所以请求都到了shardA。也就是形成了mongosA-shardB & mongoB-shardA的2个逻辑分区



问题验证与推断issue：https://jira.mongodb.org/browse/SERVER-54625

[5.0.0-rc0] 增加一个 featureFlagDefaultReadConcernLocal 参数默认为true，将默认的readConcern修改为local https://jira.mongodb.org/browse/SERVER-56488

[4.7.0] 修复了 secondary shard加载路由信息的问题 https://jira.mongodb.org/browse/SERVER-32198



---



### mongos opLatencies

Add opLatencies to mongoS serverStatus https://jira.mongodb.org/browse/SERVER-46478



---



### Opportunistic reads

4.2 版本引入feature ： **Opportunistic reads**，相关JIRA：[SERVER-41132](https://jira.mongodb.org/browse/SERVER-41132), [SERVER-41133](https://jira.mongodb.org/browse/SERVER-41133), and [SERVER-41134](https://jira.mongodb.org/browse/SERVER-41134).

特性：在mongos上进行secondary read时，从满足任意一个满足secondary来获取active的链接响应请求。

实现行为上，遍历所有secondary，通过 `ConnectionPool::get()` 来获取 connection，但是底层的实现逻辑不仅是从该host对应的Pool拿已有链接，当链接不足时还会尝试创建链接。

所以导致了 [SERVER-53853](https://jira.mongodb.org/browse/SERVER-53853) & [SERVER-53899](https://jira.mongodb.org/browse/SERVER-53899) 中 report的问题。

问题在 [SERVER-54278](https://jira.mongodb.org/browse/SERVER-54278) 被修复，修复的版本：4.2.13

修复方案：增加一个 `opportunisticSecondaryTargeting` flag，默认为 false，即不开启 Opportunistic reads，当目标节点大于1时只保留一个(`target.resize(1)`).



---



### HMAC signing keys

https://mongoing.com/archives/75350

* [4.2.7] 修复3.6版本依赖 ON_BLOCK_EXIT在析构函数里去调用appendRequiredFieldsToResponse抛出异常而导致mongos crash的问题 https://jira.mongodb.org/browse/SERVER-47553

* [4.2.10] 修复 howMuchSleepNeedFor函数中使用unsigned int溢出导致无法按时加载 signing key 的问题 https://jira.mongodb.org/browse/SERVER-48709

* [4.2.12] 修复 使用poll进行sleep时传递 timeout为负数（同样是溢出导致）导致无限睡眠无法加载signing key的问题 https://jira.mongodb.org/browse/SERVER-52654



---



### secondary 加载 路由信息 

https://docs.mongodb.com/manual/release-notes/3.6/#sharded-clusters

* [3.5.11] secondary 检查shardVersion - https://jira.mongodb.org/browse/SERVER-28948
* [3.5.10] stale secondary 触发primary加载路由信息并等待同步完成 https://jira.mongodb.org/browse/SERVER-28248
* [3.5.10] shard secondary 从本地存储加载metadata https://jira.mongodb.org/browse/SERVER-29239





