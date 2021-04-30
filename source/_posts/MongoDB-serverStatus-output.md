---
title: MongoDB serverStatus 输出指标说明
date: 2020-06-03 10:43:34
tags:
- MongoDB-serverStatus
categories:
- MongoDB
- command
- serverStatus
---



> MongoDB 4.2 serverStatus指标释义，本文只包含server层指标

### 实例信息 - Instance Information

```json
"host" : <string>,
"advisoryHostFQDNs" : <array>,
"version" : <string>,
"process" : <"mongod"|"mongos">,
"pid" : <num>,
"uptime" : <num>,
"uptimeMillis" : <num>,
"uptimeEstimate" : <num>,
"localTime" : ISODate(""),
```

* host : 
* advisoryHostFQDNs : 
* version : 
* process : 
* pid : 
* **uptime :** 
* uptimeMillis : 
* uptimeEstimate : 
* localTime : 

### * assert

自MongoDB启动以来的断言数

```json
"asserts" : {
   "regular" : <num>,
   "warning" : <num>,
   "msg" : <num>,
   "user" : <num>,
   "rollovers" : <num>
},
```

* regular : MongoDB启动以来发生的常规断言数
* warning : 4.0版本以后该值恒为0
  * 4.0之前的版本会返回MongoDB启动以来发生warning的次数
* msg : MongoDB启动以来引发的消息断言数
* user : MongoDB启动以来引发的用户断言数。这些断言可能是用户行为生成的， 比如磁盘空间不足、duplicate key等，可以通过修复应用程序或者部署问题来解决这些问题
* rollovers : MongoDB启动以来，断言的翻转次数。assert超过2^30，则会置零，rollovers+1

### * connections

当前连接数的信息

```json
"connections" : {
   "current" : <num>,
   "available" : <num>,
   "totalCreated" : <num>,
   "active" : <num>,
   "exhaustIsMaster" : <num>,
   "awaitingTopologyChanges" : <num>
},
```

* **current : 当前连接数，包含从其他rs节点或者mongos节点发起的链接**
* **available : 当前未使用的连接数**
* **totalCreated : MongoDB启动以来，创建的链接总数**
* **active : [4.0.7] 当前活跃的连接数。活跃链接：链接中有操作正在执行(in progress)**
* exhaustIsMaster : [4.4] // TODO 客户端的最后一条请求是包含 `exhaustAllowed` 的 `isMaster` 请求数
* awaitingTopologyChanges : [4.4] // TODO 当前在 `isMaster` 请求中等待拓扑变更的请求数

### defaultRWConcern [4.4] // TODO

The `defaultRWConcern` section provides information on the local copy of the global default read or write concern settings. The data may be stale or out of date. See [`getDefaultRWConcern`](https://docs.mongodb.com/master/reference/command/getDefaultRWConcern/#dbcmd.getDefaultRWConcern) for more information.

```json
"defaultRWConcern" : {
  "defaultReadConcern" : {
    "level" : <string>
  },
  "defaultWriteConcern" : {
    "w" : <string> | <int>,
    "wtimeout" : <int>,
    "j" : <bool>
  },
  "updateOpTime" : Timestamp,
  "updateWallClockTime" : Date,
  "localUpdateWallClockTime" : Date
}
```



### * electionMetrics [4.2.1 & 4.0.13]

```json
"electionMetrics" : {
   "stepUpCmd" : {
      "called" : <NumberLong>,
      "successful" : <NumberLong>
   },
   "priorityTakeover" : {
      "called" : <NumberLong>,
      "successful" : <NumberLong>
   },
   "catchUpTakeover" : {
      "called" : <NumberLong>,
      "successful" : <NumberLong>
   },
   "electionTimeout" : {
      "called" : <NumberLong>,
      "successful" : <NumberLong>
   },
   "freezeTimeout" : {
      "called" : <NumberLong>,
      "successful" : <NumberLong>
   },
   "numStepDownsCausedByHigherTerm" : <NumberLong>,
   "numCatchUps" : <NumberLong>,
   "numCatchUpsSucceeded" : <NumberLong>,
   "numCatchUpsAlreadyCaughtUp" : <NumberLong>,
   "numCatchUpsSkipped" : <NumberLong>,
   "numCatchUpsTimedOut" : <NumberLong>,
   "numCatchUpsFailedWithError" :<NumberLong>,
   "numCatchUpsFailedWithNewTerm" : <NumberLong>,
   "numCatchUpsFailedWithReplSetAbortPrimaryCatchUpCmd" : <NumberLong>,
   "averageCatchUpOps" : <double>
}
```

* stepUpCmd : primary 卸任(`stepDown`)时，mongod发起选举交接的次数
* priorityTakeover : mongod实例的 `priority` 大于 primary 时，发起选举的次数
* catchUpTakeover : mongod实例比 primary 更新时，发起选举的次数 // [`settings.catchUpTakeoverDelayMillis`](https://docs.mongodb.com/master/reference/replica-configuration/#rsconf.settings.catchUpTakeoverDelayMillis)
* electionTimeout : mongod实例在指定时间内无法连接 primary 时，发起选举的次数 // [`settings.electionTimeoutMillis`](https://docs.mongodb.com/master/reference/replica-configuration/#rsconf.settings.electionTimeoutMillis)
* freezeTimeout : 冻结时间过后，mongod发起的选举次数 // [`freeze period`](https://docs.mongodb.com/master/reference/command/replSetFreeze/#dbcmd.replSetFreeze) 
* numStepDownsCausedByHigherTerm : 发现有其他节点任期更高，mongod发生step down的次数
* numCatchUps : 作为新当选的primary，mongod必须追赶最高的oplog的次数
* numCatchUpsSucceeded : 作为新当选的primary，mongod追赶最高oplog成功的次数
* numCatchUpsAlreadyCaughtUp : 作为新当选的primary，选举时已经完成追赶而结束追赶的次数
* numCatchUpsSkipped : 作为新当选的primary，跳过追赶过程的次数
* numCatchUpsTimedOut : 作为新当选的primary，由于时间限制而结束追赶的次数 // [`settings.catchUpTimeoutMillis`](https://docs.mongodb.com/master/reference/replica-configuration/#rsconf.settings.catchUpTimeoutMillis)
* numCatchUpsFailedWithError : 作为新当选的primary，由于错误而导致追赶失败的次数
* numCatchUpsFailedWithNewTerm : 作为新当选的primary，由于其他节点有更高任期而导致追赶失败的次数
* numCatchUpsFailedWithReplSetAbortPrimaryCatchUpCmd : 作为新当选的primary，收到 [`replSetAbortPrimaryCatchUp`](https://docs.mongodb.com/master/reference/command/replSetAbortPrimaryCatchUp/#dbcmd.replSetAbortPrimaryCatchUp) 而结束追赶的次数
* averageCatchUpOps : 新当选的primary追赶过程中，平均apply的operation次数

### extra_info

```json
"extra_info" : {
   "note" : "fields vary by platform.",
   "heap_usage_bytes" : <num>,
   "page_faults" : <num>
},
```

* note : 恒为"fields vary by platform."
* heap_usage_bytes : 占用堆内存的大小
* **page_faults : 发生major page_faults的次数**
* **user_time_us :用户态CPU使用** 
* **system_time_us :系统CPU使用** 
* maximum_resident_set_kb : 
* input_blocks : 
* output_blocks : 
* **page_reclaims : 发生minor page_faults的次数**
* **voluntary_context_switches :发生自愿context switch的次数** 
* **involuntary_context_switches :发生非自愿context switch的次数** 

### flowControl [4.2]

流控相关指标

```json
"flowControl" : {
   "enabled" : <boolean>,
   "targetRateLimit" : <int>,
   "timeAcquiringMicros" : <NumberLong>,
   "locksPerKiloOp" : <double>,  // Available in 4.4+. In 4.2, returned locksPerOp instead.
   "sustainerRate" : <int>,
   "isLagged" : <boolean>,
   "isLaggedCount" : <int>,
   "isLaggedTimeMicros" : <NumberLong>,
},
```

* **enabled : 是否开启流控 // [`enableFlowControl`](https://docs.mongodb.com/master/reference/parameters/#param.enableFlowControl)**
* **targetRateLimit : primary节点返回的是每秒最大可以获取ticket的数量。secondary节点返回一个占位符**
* **timeAcquiringMicros : primary节点返回write操作的总等待时间。secondary节点返回一个占位符**
* **locksPerKiloOp : [4.4] primary节点每1000操作获取锁的近似值。secondary节点返回一个占位符**
  * **4.2 版本返回locksPerOp字段，为每个操作获取锁次数，4.4取代了该字段**
* **sustainerRate : primary节点上，返回secondary每秒apply的操作数。secondary节点返回一个占位符**
* **isLagged : 是否发生流控。当majority committed lag超过指定时间时发生。secondary节点返回一个占位符 // [`flowControlTargetLagSeconds`](https://docs.mongodb.com/master/reference/parameters/#param.flowControlTargetLagSeconds)**
* **isLaggedCount : primary节点返回mongod启动以来发生流控的次数。secondary节点返回一个占位符**
* **isLaggedTimeMicros : primary节点上返回mongod启动以来发生流控的时间。secondary返回一个占位符**

### freeMonitoring // TODO

```json
"freeMonitoring" : {
   "state" : <string>,
   "retryIntervalSecs" : <NumberLong>,
   "lastRunTime" : <string>,
   "registerErrors" : <NumberLong>,
   "metricsErrors" : <NumberLong>
},
```



### globalLock

```json
"globalLock" : {
   "totalTime" : <num>,
   "currentQueue" : {
      "total" : <num>,
      "readers" : <num>,
      "writers" : <num>
   },
   "activeClients" : {
      "total" : <num>,
      "readers" : <num>,
      "writers" : <num>
   }
},
```

* totalTime : 自mongod启动并创建 `globalLock` 的微秒数。大致和服务器uptime相同
* **currentQueue.total : 等待锁队列的长度 = `currentQueue.readers` + `currentQueue.writers`**
* **currentQueue.readers : 等待读锁队列的长度**
* **currentQueue.writers : 等待写锁队列的长度**
* **activeClients.total : 客户连接和内部threads，执行读或者写的总数。可能比 `currentQueue.readers` + `currentQueue.writers` 高，因为还包含了内部threads**
* **activeClients.readers : 客户连接执行读操作的数量**
* **activeClients.writers : 客户连接执行写操作的数量**

### hedgingMetrics [4.4] // TODO

*New in version 4.4:* For [`mongos`](https://docs.mongodb.com/master/reference/program/mongos/#bin.mongos) instances only.

```json
"hedgingMetrics" : {
   "numTotalOperations" : <num>,
   "numTotalHedgedOperations" : <num>,
   "numAdvantageouslyHedgedOperations" : <num>
},
```



### latchAnalysis [4.4] // TODO

*New in version 4.4.*

```json
"latchAnalysis" : {
   <latch name> : {
      "created" : <num>,
      "destroyed" : <num>,
      "acquired" : <num>,
      "released" : <num>,
      "contended" : <num>,
      "hierarchicalAcquisitionLevelViolations" : {
            "onAcquire" : <num>,
            "onRelease" : <num>
      }
   },
   ...
```



### logicalSessionRecordCache [3.6]

Server session 的一些缓存指标

```json
"logicalSessionRecordCache" : {
   "activeSessionsCount" : <num>,
   "sessionsCollectionJobCount" : <num>,
   "lastSessionsCollectionJobDurationMillis" : <num>,
   "lastSessionsCollectionJobTimestamp" : <Date>,
   "lastSessionsCollectionJobEntriesRefreshed" : <num>,
   "lastSessionsCollectionJobEntriesEnded" : <num>,
   "lastSessionsCollectionJobCursorsClosed" : <num>,
   "transactionReaperJobCount" : <num>,
   "lastTransactionReaperJobDurationMillis" : <num>,
   "lastTransactionReaperJobTimestamp" : <Date>,
   "lastTransactionReaperJobEntriesCleanedUp" : <num>,
   "sessionCatalogSize" : <num>   // Starting in MongoDB 4.2
},
```

* **activeSessionsCount : 上次刷新以来，缓存在内存中的本地活跃session数量**
* **sessionsCollectionJobCount : session refresh执行次数**
* **lastSessionsCollectionJobDurationMillis : 上次session refresh执行耗时**
* lastSessionsCollectionJobTimestamp : 上次session refresh发生的时间点
* **lastSessionsCollectionJobEntriesRefreshed : 上次session refresh期间，刷新的session数量**
* **lastSessionsCollectionJobEntriesEnded : 上次session refresh期间，结束的session数量**
* **lastSessionsCollectionJobCursorsClosed : 上次session refresh期间，关闭的cursor数量**
* **transactionReaperJobCount : transaction reaper执行次数**
* **lastTransactionReaperJobDurationMillis : 上次transaction reaper执行耗时**
* lastTransactionReaperJobTimestamp : 上次transaction reaper发生的时间点
* **lastTransactionReaperJobEntriesCleanedUp : 上次transaction reaper清理的entry数量**
* **sessionCatalogSize : [4.2] 内存中缓存的session数量**

### locks

```json
"locks" : {
   <type> : {
         "acquireCount" : {
            <mode> : NumberLong(<num>),
            ...
         },
         "acquireWaitCount" : {
            <mode> : NumberLong(<num>),
            ...
         },
         "timeAcquiringMicros" : {
            <mode> : NumberLong(<num>),
            ...
         },
         "deadlockCount" : {
            <mode> : NumberLong(<num>),
            ...
         }
   },
   ...
```

* **<type> :** 
  * **ParallelBatchWriterMode : [4.2]**
  * **ReplicationStateTransition : [4.2]**
  * **Global**
  * **Database**
  * **Collection**
  * **Mutex**
  * **Metadata**
  * **oplog**
* **<type>.acquireCount : 获取锁的次数**
* **<type>.acquireWaitCount : 由于锁冲突而等待的次数**
* **<type>.timeAcquiringMicros : 锁等待的累积时间**
* **<type>.deadlockCount : 死锁发生的次数**
* **<modes> :** 
  * **R : Shared (S) lock.**
  * **W : Exclusive (X) lock.**
  * **r : Intent Shared (IS) lock.**
  * **w : Intent Exclusive (IX) lock.**



### mirroredReads [4.4] // TODO

*Available on mongod only.*

```json
"mirroredReads" : {
      "seen" : <num>,
      "sent" : <num>
},
```



### network

```json
"network" : {
   "bytesIn" : <num>,
   "bytesOut" : <num>,
   "physicalBytesIn" : <num>,
	 "physicalBytesOut" : <num>,
   "numSlowDNSOperations" : <num>,
   "numSlowSSLOperations" : <num>,
   "numRequests" : <num>,
   "serviceExecutorTaskStats" : {},
   "tcpFastOpen" : {
     "kernelSetting" : NumberLong("<num>"),
     "serverSupported" : <bool>,
     "clientSupported" : <bool>,
     "accepted" : "NumberLong(<num>)"
},
```

* **bytesIn : 网络入流量的字节数**
* **bytesOut : 网络出流量字节数**
* **physicalBytesIn : 物理网络入流量**
* **physicalBytesOut : 物理网络出流量**
* numSlowDNSOperations : [4.4] DNS解析超过1s的次数
* numSlowSSLOperations : [4.4] SSL handshake超过1s的次数
* **numRequests : 接收到的请求次数**
* serviceExecutorTaskStats : 打印serviceExecutor信息，默认是 Synchronous，则信息为 `{ "executor" : "passthrough", "threadsRunning" : 2 }`
* tcpFastOpen :  [4.4]
  * kernelSetting :  [4.4] = /proc/sys/net/ipv4/tcp_fastopen
  * serverSupported :  [4.4] 服务器操作系统inbound是否支持tcpFastOpen
  * clientSupported :  [4.4] 服务器操作系统outbound是否支持tcpFastOpen
  * accepted :  [4.4] mongod实例启动以来 accepted 的tcpFastOpen次数

### opLatencies [mongod]

请求耗时信息

```json
"opLatencies" : {
   "reads" : {
			"latency" : NumberLong(0),
			"ops" : NumberLong(0)
		},
   "writes" : {
			"latency" : NumberLong(0),
			"ops" : NumberLong(0)
		},
   "commands" : {
			"latency" : NumberLong(0),
			"ops" : NumberLong(0)
		},
   "transactions" : {
			"latency" : NumberLong(0),
			"ops" : NumberLong(0)
		}
},
```

输出类似 `db.collection.aggregate( [ { $collStats: { latencyStats: { histograms: true } } } ] )`

**记录读、写、cmd、事务操作的次数和latency**

* **latency : 操作总耗时**
* **ops : 操作总执行次数**
* histogram : 记录latency的分布信息 // serverStatus中不输出

### opReadConcernCounters [4.0.6] [mongod]

```json
"opReadConcernCounters" : {
   "available" : NumberLong(<num>),
   "linearizable" : NumberLong(<num>),
   "local" : NumberLong(<num>),
   "majority" : NumberLong(<num>),
   "snapshot" : NumberLong(<num>),
   "none" :  NumberLong(<num>) // use default
}
```

mongod启动以来，query操作中指定的read concern的次数

**available / linearizable / local / majority / snapshot / none**

### opWriteConcernCounters [4.0.6] [mongod]

```json
"opWriteConcernCounters" : {
   "insert" : {
      "wmajority" : NumberLong(<num>),
      "wnum" : {
         "<num>" :  NumberLong(<num>),
         ...
      },
      "wtag" : {
         "<tag1>" :  NumberLong(<num>),
         ...
      },
      "none" : NumberLong(<num>)
   },
   "update" : {
      "wmajority" : NumberLong(<num>),
      "wnum" : {
         "<num>" :  NumberLong(<num>),
      },
      "wtag" : {
         "<tag1>" :  NumberLong(<num>),
         ...
      },
      "none" : NumberLong(<num>)
   },
   "delete" : {
      "wmajority" :  NumberLong(<num>)
      "wnum" : {
         "<num>" :  NumberLong(<num>),
         ...
      },
      "wtag" : {
         "<tag1>" :  NumberLong(<num>),
         ...
      },
      "none" : NumberLong(<num>)
   }
}
```

mongod启动以来，write操作中指定的write concern的数量。`j` 和 `wtimeout` 参数不会影响计数，即使超时也会进行计数。只有在[`reportOpWriteConcernCountersInServerStatus`](https://docs.mongodb.com/master/reference/parameters/#param.reportOpWriteConcernCountersInServerStatus)设置为true的时候才会打印，默认为false。

* **wmajority : 指定 `w: majority` 的次数**
* **wnum : 指定 `w: <num>` 的次数**
* **wtag : 指定 `w: <tag>` 的次数**
* **none : 未指定 `w` 的次数，这些操作使用默认 `w: 1`**

### opcounters

```json
"opcounters" : {
   "insert" : NumberLong(<num>),  // Starting in MongoDB 4.2, type is NumberLong
   "query" : NumberLong(<num>),   // Starting in MongoDB 4.2, type is NumberLong
   "update" : NumberLong(<num>),  // Starting in MongoDB 4.2, type is NumberLong
   "delete" : NumberLong(<num>),  // Starting in MongoDB 4.2, type is NumberLong
   "getmore" : NumberLong(<num>), // Starting in MongoDB 4.2, type is NumberLong
   "command" : NumberLong(<num>), // Starting in MongoDB 4.2, type is NumberLong
},
```

记录启动以来各个命令的执行次数

**insert / query / update / delete / getmore / command**

4.2版本开始，这些指标的类型修改为NumberLong，之前版本为NumberInt。

### opcountersRepl

```json
"opcountersRepl" : {
   "insert" : NumberLong(<num>),  // Starting in MongoDB 4.2, type is NumberLong
   "query" : NumberLong(<num>),   // Starting in MongoDB 4.2, type is NumberLong
   "update" : NumberLong(<num>),  // Starting in MongoDB 4.2, type is NumberLong
   "delete" : NumberLong(<num>),  // Starting in MongoDB 4.2, type is NumberLong
   "getmore" : NumberLong(<num>), // Starting in MongoDB 4.2, type is NumberLong
   "command" : NumberLong(<num>), // Starting in MongoDB 4.2, type is NumberLong
},
```

记录启动以来replication操作的次数

**insert / query / update / delete / getmore / command**

4.2版本开始，这些指标的类型修改为NumberLong，之前版本为NumberInt。只有当前实例为`replica set`的成员的时候才会打印。这些值和 `opcounter` 中的值可能不同，因为mongod可能序列化了这些操作

### oplogTruncation [4.2.1]

```json
"oplogTruncation" : {
   "totalTimeProcessingMicros" : <NumberLong>,
   "processingMethod" : <string>,
   "oplogMinRetentionHours" : <double>
   "totalTimeTruncatingMicros" : <NumberLong>,
   "truncateCount" : <NumberLong>
},
```



### repl

```json
"repl" : {
   "hosts" : [
         <string>,
         <string>,
         <string>
   ],
   "setName" : <string>,
   "setVersion" : <num>,
   "ismaster" : <boolean>,
   "secondary" : <boolean>,
   "primary" : <hostname>,
   "me" : <hostname>,
   "electionId" : ObjectId(""),
   "rbid" : <num>,
   "replicationProgress" : [
         {
            "rid" : <ObjectId>,
            "optime" : { ts: <timestamp>, term: <num> },
            "host" : <hostname>,
            "memberId" : <num>
         },
        ...
   ]
}
```

* hosts : 当前 `replica set` 成员的 `host:port` 的数组
* setName :  当前 `replica set` 的name
* setVersion :  // TODO
* ismaster : 当前是否为 primary 节点
* secondary : 当前是否为 secondary 节点
* primary : primary节点的 `host:port` 信息
* me : 当前节点的 `host:port` 信息
* electionId :  // TODO 
* rbid : rollback标识符，表示当前节点是否发生了rollback
* lastWrite : 
  * **opTime :** 
  * lastWriteDate : 
  * **majorityOpTime :** 
  * majorityWriteDate : 
* replicationProgress : 一个数组，每个成员是一个向当前节点汇报复制状态的节点。如果要包含这部分信息，需要在 `serverStatus` 中增加 `repl` 的option
  * rid : `replica set`成员使用的ID，仅内部使用
  * **optime : 该成员汇报的最新apply的oplog的时间**
  * host : 该成员的 `host:port` 信息
  * memberId : 该成员的整数标识符

### security

```json
"security" : {
   "authentication" : {
      "mechanisms" : {
         "MONGODB-X509" : {
            "speculativeAuthenticate" : {
               "received" : <num>,
               "successful" : <num>
            },
            "authenticate" : {
               "received" : <num>,
               "successful" : <num>
            }
         },
         "SCRAM-SHA-1" : {
            "speculativeAuthenticate" : {
               "received" : <num>,
               "successful" : <num>
            },
            "authenticate" : {
               "received" : <num>,
               "successful" : <num>
            }
         },
         "SCRAM-SHA-256" : {
            "speculativeAuthenticate" : {
               "received" : <num>,
               "successful" : <num>
            },
            "authenticate" : {
               "received" : <num>,
               "successful" : <num>
            }
          }
       }
     },
     "SSLServerSubjectName": <string>,
     "SSLServerHasCertificateAuthority": <boolean>,
     "SSLServerCertificateExpirationDate": <date>
},
```

* // TODO 4.4 new metrics
* SSLServerSubjectName : SSL证书的subject name
* SSLServerHasCertificateAuthority : 与相关机构关联时为true，自签名的为false
* SSLServerCertificateExpirationDate : SSL证书的过期时间

### sharding

```json
{
   "configsvrConnectionString" : "csRS/cfg1.example.net:27019,cfg2.example.net:27019,cfg2.example.net:27019",
   "lastSeenConfigServerOpTime" : {
      "ts" : Timestamp(1517462189, 1),
      "t" : NumberLong(1)
   },
   "maxChunkSizeInBytes" : NumberLong(67108864)
}
```

* configsvrConnectionString : config server的连接串
* lastSeenConfigServerOpTime : mongos 或者 shard获取的最新 CSRS 的 optime
* maxChunkSizeInBytes : [3.6] 配置的最大chunk size。非实时

3.2版本，mongos返回sharding信息。3.6版本，shard返回sharding信息

### shardingStatistics [4.0]

#### run on shard

```json
"shardingStatistics" : {
   "countStaleConfigErrors" : NumberLong(<num>),
   "countDonorMoveChunkStarted" : NumberLong(<num>),
   "totalDonorChunkCloneTimeMillis" : NumberLong(<num>),
   "totalCriticalSectionCommitTimeMillis" : NumberLong(<num>),
   "totalCriticalSectionTimeMillis" : NumberLong(<num>),
   "countDocsClonedOnRecipient" : NumberLong(<num>),
   "countDocsClonedOnDonor" : NumberLong(<num>),
   "countRecipientMoveChunkStarted" : NumberLong(<num>),
   "countDocsDeletedOnDonor" : NumberLong(<num>),
   "countDonorMoveChunkLockTimeout" : NumberLong(<num>),
   "unfinishedMigrationFromPreviousPrimary" : NumberLong(<num>),
   "catalogCache" : {
      "numDatabaseEntries" : NumberLong(<num>),
      "numCollectionEntries" : NumberLong(<num>),
      "countStaleConfigErrors" : NumberLong(<num>),
      "totalRefreshWaitTimeMicros" : NumberLong(<num>),
      "numActiveIncrementalRefreshes" : NumberLong(<num>),
      "countIncrementalRefreshesStarted" : NumberLong(<num>),
      "numActiveFullRefreshes" : NumberLong(<num>),
      "countFullRefreshesStarted" : NumberLong(<num>),
      "countFailedRefreshes" : NumberLong(<num>)
   },
   "rangeDeleterTasks" : <num>
},
```

#### run on mongos

```json
"shardingStatistics" : {
   "catalogCache" : {
      "numDatabaseEntries" : NumberLong(<num>),
      "numCollectionEntries" : NumberLong(<num>),
      "countStaleConfigErrors" : NumberLong(<num>),
      "totalRefreshWaitTimeMicros" : NumberLong(<num>),
      "numActiveIncrementalRefreshes" : NumberLong(<num>),
      "countIncrementalRefreshesStarted" : NumberLong(<num>),
      "numActiveFullRefreshes" : NumberLong(<num>),
      "countFullRefreshesStarted" : NumberLong(<num>),
      "countFailedRefreshes" : NumberLong(<num>),
      "operationsBlockedByRefresh" : {
        "countAllOperations" : NumberLong(<num>),
        "countInserts" : NumberLong(<num>),
        "countQueries" : NumberLong(<num>),
        "countUpdates" : NumberLong(<num>),
        "countDeletes" : NumberLong(<num>),
        "countCommands" : NumberLong(<num>)
      }
   }
},
```

* **countStaleConfigErrors : [mongod] 触发 stale config的次数**
* **countDonorMoveChunkStarted : [mongod] 从当前实例发起 `moveChunk` 的次数**
* **totalDonorChunkCloneTimeMillis : [mongod] 从当前实例进行 `moveChunk` 过程中，clone阶段总耗时**
* **totalCriticalSectionCommitTimeMillis : [mongod] 从当前实例进行 `moveChunk` 过程中，更新元数据的总耗时**
* **totalCriticalSectionTimeMillis : [mongod] 从当前实例进行 `moveChunk` 过程中，追赶和更新元数据的总耗时**
  * **追赶阶段耗时 = `totalCriticalSectionTimeMillis` - `totalCriticalSectionCommitTimeMillis`**
* **countDocsClonedOnRecipient : [mongod4.2] 作为 `moveChunk` 接收方，clone的document总数**
* **countDocsClonedOnDonor : [mongod4.2] 作为 `moveChunk` 发送方，clone的document总数**
* **countRecipientMoveChunkStarted : [mongod4.2] 作为 `moveChunk` 接收方，开始接收的次数**
* **countDocsDeletedOnDonor : [mongod4.2] 作为 `moveChunk` 发送方，已经删除的document总数**
* **countDonorMoveChunkLockTimeout : [mongod4.2] 作为 `moveChunk` 发送方，因获取锁超时而中断的次数**
* **unfinishedMigrationFromPreviousPrimary : [mongod4.4] 因为选主导致migration未完成的次数，只有选主完成才会更新**
* **rangeDeleterTasks : [mongod4.4] migration之后准备range delete的任务总数**
* **catalogCache :** 
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

### shardedIndexConsistency [4.2.6 & 4.4] [config]

```json
"shardedIndexConsistency" : {
   "numShardedCollectionsWithInconsistentIndexes" : <NumberLong>
},
```

* **numShardedCollectionsWithInconsistentIndexes : 分片中索引不一致的分片集合数**
  * [`enableShardedIndexConsistencyCheck`](https://docs.mongodb.com/master/reference/parameters/#param.enableShardedIndexConsistencyCheck)
  * [`shardedIndexConsistencyCheckIntervalMS`](https://docs.mongodb.com/master/reference/parameters/#param.shardedIndexConsistencyCheckIntervalMS)

### storageEngine

```json
"storageEngine" : {
   "name" : <string>,
   "supportsCommittedReads" : <boolean>,
   "oldestRequiredTimestampForCrashRecovery" : Timestamp(0, 0),
   "supportsPendingDrops" : true,
   "dropPendingIdents" : NumberLong(0),
   "supportsSnapshotReadConcern" : true,
   "readOnly" : false,
   "persistent" : <boolean>
   "backupCursorOpen" : false
},
```

* name : 当前存储引擎的名称
* supportsCommittedReads : [3.2] 当前存储引擎是否支持 `majority read concern`
* persistent : [3.2.6] 是否支持持久化到磁盘

### transactions

#### run on mongod [3.6.3]

```json
"transactions" : {
   "retriedCommandsCount" : <NumberLong>,
   "retriedStatementsCount" : <NumberLong>,
   "transactionsCollectionWriteCount" : <NumberLong>,
   "currentActive" : <NumberLong>,
   "currentInactive" : <NumberLong>,
   "currentOpen" : <NumberLong>,
   "totalAborted" : <NumberLong>,
   "totalCommitted" : <NumberLong>,
   "totalStarted" : <NumberLong>,
   "totalPrepared" : <NumberLong>,
   "totalPreparedThenCommitted" : <NumberLong>,
   "totalPreparedThenAborted" :  <NumberLong>,
   "currentPrepared" :  <NumberLong>,
   "lastCommittedTransaction" : <document> // Starting in 4.2.2 (and 4.0.9)
},
```

#### run on mongos [4.2]

```json
   "currentOpen" : <NumberLong>,     // Starting in 4.2.1
   "currentActive" : <NumberLong>,   // Starting in 4.2.1
   "currentInactive" : <NumberLong>, // Starting in 4.2.1
   "totalStarted" : <NumberLong>,
   "totalCommitted" : <NumberLong>,
   "totalAborted" : <NumberLong>,
   "abortCause" : {
      <String1> : <NumberLong>,
      <String2>" : <NumberLong>,
      ...
   },
   "totalContactedParticipants" : <NumberLong>,
   "totalParticipantsAtCommit" : <NumberLong>,
   "totalRequestsTargeted" : <NumberLong>,
   "commitTypes" : {
      "noShards" : {
         "initiated" : <NumberLong>,
         "successful" : <NumberLong>,
         "successfulDurationMicros" : <NumberLong>,
      },
      "singleShard" : {
         "initiated" : <NumberLong>,
         "successful" : <NumberLong>,
         "successfulDurationMicros" : <NumberLong>,
      },
      "singleWriteShard" : {
         "initiated" : <NumberLong>,
         "successful" : <NumberLong>,
         "successfulDurationMicros" : <NumberLong>,
      },
      "readOnly" : {
         "initiated" : <NumberLong>,
         "successful" : <NumberLong>,
         "successfulDurationMicros" : <NumberLong>,
      },
      "twoPhaseCommit" : {
         "initiated" : <NumberLong>,
         "successful" : <NumberLong>,
         "successfulDurationMicros" :<NumberLong>,
      },
      "recoverWithToken" : {
         "initiated" : <NumberLong>,
         "successful" : <NumberLong>,
         "successfulDurationMicros" : <NumberLong>,
      }
   }
},
```

* **retriedCommandsCount : [mongod3.6.3] 在可重入写操作提交后，总的重试次数**
* **retriedStatementsCount : [mongod3.6.3] retriedCommandsCount中涉及的statment总数**
* **transactionsCollectionWriteCount : [mongod3.6.3] 当可重入写操作提交后，触发写入config.transaction次数**
* **currentActive : [mongod4.0.2 & mongos4.2.1] 当前活跃的transaction数**
* **currentInactive : [mongod4.0.2 & mongos4.2.1] 当前不活跃的事务数**
* **currentOpen : [mongod4.0.2 & mongos4.2.1] 当前open的事务数**
* **currentPrepared : [mongod4.2] 当前prepare的事务数**
* **totalPrepared : [mongod4.2] 启动以来prepare的事务数**
* **totalPreparedThenCommitted : [mongod4.2] 启动以来prepare然后commit的事务数**
* **totalPreparedThenAborted : [mongod4.2] 启动以来prepare然后abort的事务数**
* **totalStarted : [mongod4.0.2 & mongos4.2] 启动以来总的start的事务数**
* **totalCommitted : [mongod4.0.2 & mongos4.2] 启动以来总的commit事务数**
* **totalAborted : [mongod4.0.2 & mongos4.2] 启动以来总的abort事务数**
* **abortCause : [mongos4.2] abort的原因及数量。abort / DuplicateKey / StaleConfig / SnapshotTooOld 等**
* lastCommittedTransaction : [mongod4.2.2 & 4.0.9] primary节点汇报上次commit事务的信息，secondary节点汇报曾经是primary时上次提交的信息
  * operationCount : 事务中的写操作数
  * oplogOperationBytes : 事务相应oplog大小
  * writeConcern : 事务的write concern
* **totalContactedParticipants ：[mongos4.2] 启动以来因事务联系的shard总数**
* **totalParticipantsAtCommit ： [mongos4.2] 启动以来事务提交涉及的shard总数**
* **totalRequestsTargeted ： [mongos4.2] 启动以来mongos发起的事务请求总数**
* **commitTypes ： [mongos4.2] 提交类型**
  * **<type>**
    * **noShards : commit不需要联系任何shard**
    * **singleShard : commit只影响单个shard**
    * **singleWriteShard : commit影响多个shard，但是写操作只影响一个shard**
    * **readOnly : 只读事务**
    * **twoPhaseCommit : commit包含多个shard的写入操作**
    * **recoverWithToken : 在另一个实例或者当前实例重启后恢复的提交的事务数**
  * **<metric>**
    * **initiated : 发起该类commit的总数**
    * **successful : 该类commit成功的总数**
    * **successfulDurationMicros : 该类commit成功提交的总耗时**

### transportSecurity [4.0.2 & 3.6.7+ & 3.4.17+]

```json
"transportSecurity" : {
   "1.0" : <NumberLong>,
   "1.1" : <NumberLong>,
   "1.2" : <NumberLong>,
   "1.3" : <NumberLong>,
   "unknown" :<NumberLong>
},
```

mongod启动以来，使用对应TLS版本的连接数

### writeBacksQueued // TODO

```json
"writeBacksQueued" : <boolean>,
```



### mem

```json
"mem" : {
   "bits" : <int>,
   "resident" : <int>,
   "virtual" : <int>,
   "supported" : <boolean>,
   "mapped" : <int>,
   "mappedWithJournal" : <int>
},
```

* bits : `64` or `32` , 标识实例是被64位还是32位编译的
* **resident : resident memory, MB**
* **virtual : virtual memory, MB**
* supported : 标识是否支持扩展内存信息
* mapped : supported = false时的补充说明信息
* mappedWithJournal : 

### metrics

```json
"metrics" : {
   "aggStageCounters" : {
         "<aggregation stage>" : <num>
         }
   },
   "commands": {
         "<command>": {
            "failed": <num>,
            "total": <num>
         }
   },
   "cursor" : {
         "timedOut" : NumberLong(<num>),
         "open" : {
            "noTimeout" : NumberLong(<num>),
            "pinned" : NumberLong(<num>),
            "multiTarget" : NumberLong(<num>),
            "singleTarget" : NumberLong(<num>),
            "total" : NumberLong(<num>),
         }
   },
   "document" : {
         "deleted" : NumberLong(<num>),
         "inserted" : NumberLong(<num>),
         "returned" : NumberLong(<num>),
         "updated" : NumberLong(<num>)
   },
   "getLastError" : {
         "wtime" : {
            "num" : <num>,
            "totalMillis" : <num>
         },
         "wtimeouts" : NumberLong(<num>),
         "default" : {
             "unsatisfiable" : <num>
             "wtimeouts" : <num>
         }
   },
   "operation" : {
         "scanAndOrder" : NumberLong(<num>),
         "writeConflicts" : NumberLong(<num>)
   },
   "queryExecutor": {
         "scanned" : NumberLong(<num>),
         "scannedObjects" : NumberLong(<num>)
   },
   "record" : {
         "moves" : NumberLong(<num>)
   },
   "repl" : {
      "executor" : {
         "pool" : {
            "inProgressCount" : <num>
         },
         "queues" : {
            "networkInProgress" : <num>,
            "sleepers" : <num>
         },
         "unsignaledEvents" : <num>,
         "shuttingDown" : <boolean>,
         "networkInterface" : <string>
      },
      "apply" : {
         "attemptsToBecomeSecondary" : <NumberLong>,
         "batches" : {
            "num" : <num>,
            "totalMillis" : <num>
         },
         "ops" : <NumberLong>
      },
      "buffer" : {
         "count" : <NumberLong>,
         "maxSizeBytes" : <NumberLong>,
         "sizeBytes" : <NumberLong>
      },
      "initialSync" : {
         "completed" : <NumberLong>,
         "failedAttempts" : <NumberLong>,
         "failures" : <NumberLong>,
      },
      "network" : {
         "bytes" : <NumberLong>,
         "getmores" : {
            "num" : <num>,
            "totalMillis" : <num>
         },
         "notMasterLegacyUnacknowledgedWrites" : <NumberLong>,
         "notMasterUnacknowledgedWrites" : <NumberLong>,
         "ops" : <NumberLong>,
         "readersCreated" : <NumberLong>,
         "replSetUpdatePosition" : {
             "num" : <NumberLong>
         }
      },
      "stepDown" : {
         "userOperationsKilled" : <NumberLong>,
         "userOperationsRunning" : <NumberLong>
      },
      "syncSource" : {
         "numSelections" : <NumberLong>,
         "numTimesChoseSame" : <NumberLong>,
         "numTimesChoseDifferent" : <NumberLong>,
         "numTimesCouldNotFind" : <NumberLong>
      }
   },
   "storage" : {
         "freelist" : {
            "search" : {
               "bucketExhausted" : <num>,
               "requests" : <num>,
               "scanned" : <num>
            }
         }
   },
   "ttl" : {
         "deletedDocuments" : NumberLong(<num>),
         "passes" : NumberLong(<num>)
   }
},
```

* aggStageCounters : [4.4 & 4.2.6 & 4.0.19] 使用 `aggregation pipeline stage` 各个stage的执行次数
* **commands :**
* **<command>.failed : 启动以来<command> fail次数**
  * **<command>.total : 启动以来<command> 执行次数**
* cursor : 
     * **timedOut : cursor因timeout关闭的数量**
    * **open.noTimeout : open cursor中设置了 `noTimeOut`的数量**
    * **open.pinned : open cursor中 pinned的数量**
    * **open.multiTarget : [mongos] 仅指向到一个shard的cursor数量**
    * **open.singleTarget : [mongos] 指向到多余1个shard的cursor数量**
    * **open.total : 维护的open cursor总数**
* **document : 文档访问或者修改的数量。[deleted / inserted / returned / updated]**
* getLastError : 
     * wtime.num : `getLastError`操作 w > 1 的次数
     * wtime.totalMillis : `getLastError`操作 w > 1 的时间
     * wtimeouts : `getLastError` 触发 `wtimeout`的次数
     * default : 
         * unsatisfiable : 返回unsatisfiable的次数
         * wtimeouts : 返回wtimeouts的次数
* operation : 
     * **scanAndOrder : 无法使用index进行sort的请求数**
     * **writeConflicts : 发生write conflict的次数**
 * queryExecutor ：
   * **scanned : 查询和查询计划中评估扫描索引项的数量， = `explain().totalKeysExamined`**
   * **scannedObjects : 查询和查询计划中评估扫描document的数量， = `explain().totalDocsExamined`**
* query

- * **planCacheTotalSizeEstimateBytes : plan cache占用内存量**

- * **updateOneOpStyleBroadcastWithExactIDCount : update请求的{multi:false} ， 但是需要broadcast去获取匹配的_id信息**

- repl : 

  * executor : ReplicationCoordinator使用的TaskExecutor的信息
      * pool.inProgressCount : 执行队列长度
      * queues.networkInProgress : 等待网络线程返回结果队列长度
      * queues.sleepers : sleep队列长度
      * unsignaledEvents : 阻塞时间列表
      * shuttingDown : 是否shutdown
      * networkInterface : 简单说明 // DEPRECATED: getDiagnosticString is deprecated in NetworkInterfaceTL
  * apply : 记录从replication中apply的oplog信息
      * **batchSize : [4.0.6 & 3.6.11] 启动以来apply的oplog总数，一批apply以后会增加批的总操作数**
      * attemptsToBecomeSecondary : 
      * **batches.num : 接收的batch数**
      * **batches.totalMillis : apply oplog花费的总时间**
      * **ops : 启动以来apply的oplog总数，每次apply都会增加**
  * buffer : 
      * **count : oplog buffer中operation的数量**
      * maxSizeBytes : oplog buffer的最大值
      * sizeBytes : 当前oplog buffer的大小
  * initialSync : 
      * **completed : initialSync成功的次数**
      * **failedAttempts : initialSync 失败重试的次数**
      * **failures : initialSync 最终失败的次数**
  * network : 
      * **bytes : 从复制源读取到的数据总数**
      * **getmores.num : 从复制源请求其他操作集合的getMore总数**
      * **getmores.totalMillis : getMore总耗时**
      * getmores.numEmptyBatches : [4.4] getMore从复制源获取到空oplog的总数
      * notMasterLegacyUnacknowledgedWrites : [4.2] 由于当前节点未进入primary状态，导致未确认的legacy写操作失败数
      * notMasterUnacknowledgedWrites : [4.2] 由于当前节点未进入primary状态，导致未确认的写操作失败数
      * **ops : 从复制源读取到的总ops数**
      * readersCreated : 创建oplog查询进程总数。当链接发生错误时会进行创建，修改数据源也会进行创建
      * replSetUpdatePosition.num : [4.4] 发送给复制源 `replSetUpdatePosition` 数量
  * stepDown : [4.2] 
      * **userOperationsKilled : [4.2] stepDown时kill的用户操作数**
      * **userOperationsRunning : [4.2] stepDown时仍然保留的用户操作数**
  * syncSource : [4.4]
      * **numSelections : [4.4] 节点重新从可用复制源选择复制源的总数**
      * **numTimesChoseSame : [4.4] 重新评估复制源时仍然选择相同复制源的次数**
      * **numTimesChoseDifferent : [4.4]  重新评估复制源时仍然选择不同复制源的次数**
      * **numTimesCouldNotFind : [4.4] 尝试选择同步源却无法找到的次数**

- storage : 

   * freelist.search.bucketExhausted : 从freelist中查找未找到适合大小的空间次数
   * freelist.search.requests : 搜索可用空间次数
   * freelist.search.scanned : 搜索到可用空间次数

 - ttl : 

    * **deletedDocuments : ttl索引删除的文档数**
    * **passes : 后台ttl线程进行删除的次数**

### watchdog [3.6] // TODO

```json
"watchdog" : {
   "checkGeneration" : NumberLong(<num>),
   "monitorGeneration" : NumberLong(<num>),
   "monitorPeriod" : <num>
}
```



### tcmalloc

记录tcmalloc占用的内存指标，方便内存分析。

*  **pageheap_free_bytes : heap中空闲的内存**

* **total_free_bytes = = `central_cache_free_bytes + transfer_cache_free_bytes + thread_cache_free_bytes` , 不包括pageheap_free_bytes**

* **central_cache_free_bytes**

* **transfer_cache_free_bytes**

* **thread_cache_free_bytes**