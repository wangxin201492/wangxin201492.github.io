---
title: MongoDB 请求计数逻辑
date: 2020-04-22 17:25:43
tags:
- MongoDB-OTT
categories:
- MongoDB
- command
- serverStatus
---

mongos上一个find请求堆栈主要路径是这个样子的：

```
#0 mongo::(anonymous namespace)::ClusterFindCmd::Invocation::run  at src/mongo/s/commands/cluster_find_cmd.cpp:205

#1 0x00007f3701fae26a in mongo::(anonymous namespace)::execCommandClient  at src/mongo/s/commands/strategy.cpp:303

#2 0x00007f3701faff38 in mongo::(anonymous namespace)::runCommand  at src/mongo/s/commands/strategy.cpp:489

#3 0x00007f3701fb1f76 in mongo::Strategy::clientCommand at src/mongo/s/commands/strategy.cpp:800

#4 0x00007f3701e243cc in mongo::ServiceEntryPointMongos::handleRequest  at src/mongo/s/service_entry_point_mongos.cpp:95

#5 0x00007f3701e7d5e2 in mongo::ServiceStateMachine::_processMessage  at src/mongo/transport/service_state_machine.cpp:452
```



`db.serverStatus()` 输出结果中有3个地方对请求进行了计数，这里简单整理下

1. `db.serverStatus().network.numRequests`(简称**requests计数**） -- 对应在堆栈中#5（即 `ServiceStateMachine::_processMessage()`）进行计数。表示**接收的网络包数量**
2. `db.serverStatus().metrics.commands.*.total & failed`（简称**metrics计数**） -- 对应在堆栈中#1（即 `execCommandClient()` ）进行计数。表示**不同命令执行的次数**
3. `db.serverStatus().opcounters`（简称**opcounters计数**） -- 对应在堆栈中#0（即 `Invocation::run()`）即进行计数。表示**requests计数**

**requests计数**和**metrics计数**的逻辑是固定的，而对于opcounters计数会根据请求的不同有一定gap。

## opcounters 计数分析

`db.serverStatus().opcounters` 包含如下结果：

```json
mongos> db.serverStatus().opcounters
{
        "insert" : NumberLong(6),
        "query" : NumberLong(1),
        "update" : NumberLong(2),
        "delete" : NumberLong(0),
        "getmore" : NumberLong(0),
        "command" : NumberLong(75)
}
```



在`ServiceEntryPointMongos::handleRequest()`中对计数场景进行了区分：

### 计数场景1：(`opcode == OP_MSG`) 或者 (`opcode == OP_QUERY` 且 namespace的collection部分 == "$cmd")

按照上面举例的find堆栈 `clientCommand()` --> `runCommand()` --> `execCommandClient()` --> `Invocation::run()` 依次调用。这里所有的command都会继承 Command 类，该类提供一个 `shouldAffectCommandCounter()` 返回True。

- insert / update / delete / query / getmore 请求重写了 `shouldAffectCommandCounter()` 返回 false ，在 `Invocation::run()` 中完成计数
- 其余请求均作为 command，在 `execCommandClient()` 中完成计数

 

### 计数场景2：`opcode == OP_QUERY` 且 namespace的collection部分 != "$cmd"

直接调用 `Strategy::queryOp()`， 并在其中完成计数

 

### 计数场景3：`opcode == OP_GET_MORE`

直接调用 `Strategy::getMore()`，并在其中完成计数

 

### 计数场景4：`opcode == OP_KILL_CURSORS`

直接调用 `Strategy::killCursors()`，并在其中调用 `OpCounters::gotOp()`

gotOp 对 OP_KILL_CURSORS / OP_REPLY 不进行计数，即这两种场景不会记录在 opcounters 中

 

### 计数场景5：`opcode == OP_INSERT / OP_UPDATE / OP_DELETE`

按照场景1堆栈 `Strategy::writeOp()` --> `runCommand()` --> `execCommandClient()` --> `Invocation::run()` 依次调用，同场景1

 

### 对`insert / update / delete` 请求的计数说明

对于insert / update / delete 请求，会在 `Invocation::run()` 中完成计数，但是对这3个请求的处理不是简单的+1完成的。增加的数量为 `batchedRequest.sizeWriteOps()`。由于3.6版本中新增了 OP_MSG 的协议类型，支持这3种请求携带多条数据：即 OP_MSG 协议中 **kind=Document Sequence 的 sections 字段**。这3种请求的具体实现，都重写了 Command 类提供的 parse 函数，基于原有的 request 构建了一个额外的 `BatchedCommandRequest` ，调用的是 `BatchedCommandRequest::parseInsert()` 函数。

 

实际执行验证来看：

通过insert插入一个BSONArray，opcounters.insert 会增加 array 的 size。和上面的分析是对齐的

```json
mongos> db.serverStatus().opcounters
{
	"insert" : NumberLong(2),
	"query" : NumberLong(0),
	"update" : NumberLong(0),
	"delete" : NumberLong(0),
	"getmore" : NumberLong(0),
	"command" : NumberLong(58)
}
mongos> db.collection.insert([{name: "B"},{name: "C"}])
BulkWriteResult({
	"writeErrors" : [ ],
	"writeConcernErrors" : [ ],
	"nInserted" : 2,
	"nUpserted" : 0,
	"nMatched" : 0,
	"nModified" : 0,
	"nRemoved" : 0,
	"upserted" : [ ]
})
mongos> db.serverStatus().opcounters
{
	"insert" : NumberLong(4),
	"query" : NumberLong(0),
	"update" : NumberLong(0),
	"delete" : NumberLong(0),
	"getmore" : NumberLong(0),
	"command" : NumberLong(62)
} 
```



而 `db.colleciton.update()` / `db.colleciton.delete()` 并不像 `insert()` 一样支持多条记录的修改。这几个函数都是被 mongo shell 封装过的，具体通过 insert / update / delete 进行多条document修改可以参考[官网文档](https://docs.mongodb.com/manual/reference/command/nav-crud/)中提供的命令，分别对应有 inserts / updates / deletes 字段支持多条document修改



**Notice:** 按照上面的分析，对于opcode == OP_QUERY / OP_GET_MORE / OP_KILL_CURSORS 的场景，对metrics计数应该是有缺失的。