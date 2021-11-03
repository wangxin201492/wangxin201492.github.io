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





