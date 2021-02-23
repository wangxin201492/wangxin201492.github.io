---
title: MongoDB中 ServiceContext/Client/OperationContext的关系
date: 2020-08-03 11:03:12
tags:
- MongoDB-OTT
categories:
- MongoDB
---

## ServiceContext

```c++
/**
 * Class representing the context of a service, such as a MongoD database service or
 * a MongoS routing service.
 *
 * A ServiceContext is the root of a hierarchy of contexts.  A ServiceContext owns
 * zero or more Clients, which in turn each own OperationContexts.
 */
class ServiceContext final : public Decorable<ServiceContext> {}
```

一个mongod or mongos 进程中只有一个全局唯一的 `ServiceContext`：`ServiceContext` 代表一个service的上下文环境，一个 `ServiceContext` 是上下文层次中的根。一个 `ServiceContext` 拥有0个或多个 `Client`， 一个 `Client` 拥有各自的 `OperationContext`

## Client

```c++
/**
 * The database's concept of an outside "client".
 * */
class Client final : public Decorable<Client> {}
```

一个外部链接对应一个 `Client`

## OperationContext

```c++
/**
 * This class encompasses the state required by an operation and lives from the time a network
 * operation is dispatched until its execution is finished. Note that each "getmore" on a cursor
 * is a separate operation. On construction, an OperationContext associates itself with the
 * current client, and only on destruction it deassociates itself. At any time a client can be
 * associated with at most one OperationContext. Each OperationContext has a RecoveryUnit
 * associated with it, though the lifetime is not necesarily the same, see releaseRecoveryUnit
 * and setRecoveryUnit. The operation context also keeps track of some transaction state
 * (RecoveryUnitState) to reduce complexity and duplication in the storage-engine specific
 * RecoveryUnit and to allow better invariant checking.
 */
class OperationContext : public Interruptible, public Decorable<OperationContext> {}
```

该类包含了一个 operation 所需的所有状态，生命周期 从 网络分派 开始，到 执行完成 为止。`cursor` 上的每一个 `getmore` 都是一个独立的 operation。在 构造函数 中，一个 `OperationContext` 会被关联到当前的 `Client` 中，直到被销毁时才会取消关联。任何时间，每个 `Client` 最多只会被关联到一个 `OperationContext` 上。每个 `OperationContext` 也会持有 `RecoveryUnit`/`RecoveryUnitState` 等对象



## 源码梳理

从源码上看，mongos 和 mongod 启动的时候都会通过 `setGlobalServiceContext(ServiceContext::make());` 来初始化一个 `ServiceContext`，这里最终将初始化好的 `ServiceContext` 放到 `globalServiceContext` 这个全局变量里，并提供一个 `getGlobalServiceContext()` 函数方便对象获取。



每有一个客户端连接到 mongod/mongos 时，会同时创建一个 `ServiceStateMachine` 用于管理该客户端连接生命周期中的状态，在 `ServiceStateMachine` 构造函数中，会调用 `ServiceContext->makeClient()` 来创建一个 `Client`对象，`makeClient()` 同时也会将创建好的 `Client` 对象添加到 `ServiceContext` 维护的 `stdx::unordered_set<Client*> _clients` 中。



在 `ServiceStateMachine` 对 `Client` 请求进行状态管理时，`Process` 为请求处理的状态，对该状态的处理在 `ServiceStateMachine::_processMessage()` 完成。这个方法中通过 `Client::getCurrent()->makeOperationContext();` 来构建一个 `OperationContext`，并在方法结束前将 `OperationContext` reset掉。



上面过程会涉及到2个自增的ID，这里简单说下：

* 每有客户端连接时，会创建 `Session` 对象，基于这个 `Session` 对象创建上面提及的 `ServiceStateMachine`。`Session` 创建的时候则会通过 `AtomicWord<unsigned long long> sessionIdCounter` 来拿到一个唯一自增ID。mongod/mongos 会创建唯一的线程来处理 `Client` 的请求，该线程名称 `conn123` 中的 123 即是 上面获取到的sessionId
* `ServiceContext::makeOperationContext()` 创建 `OperationContext` 时会传入一个id用于唯一标识该 `OperationContext`，这个ID的获取是通过 `ServiceContext` 维护的 `AtomicWord<unsigned> _nextOpId` 拿到的。这个 ID 即为 `killOp`/`currentOp` 等命令中展示的 `opid`



同时 `ServiceContext` 提供了一个 `ClientObserver` 类用于 `Client` / `OperationContext` 构造和析构时的hook。

```c++
    /**
     * Observer interface implemented to hook client and operation context creation and
     * destruction.
     */
    class ClientObserver {
    public:
        ...
        virtual void onCreateClient(Client* client) = 0;
        virtual void onDestroyClient(Client* client) = 0;
        virtual void onCreateOperationContext(OperationContext* opCtx) = 0;
        virtual void onDestroyOperationContext(OperationContext* opCtx) = 0;
    };
```

`ClientObserver` 的子类实现上述 virtual 函数，并通过 `ServiceContext::registerClientObserver()` 将自身注册到 `ServiceContext` 的 `std::vector<ClientObserverHolder> _clientObservers;` 中。那么在上文描述到的创建 `Client` 的函数 `ServiceContext::makeClient()` 和创建 `OperationContext` 函数 `ServiceContext::makeOperationContext()` 中则会依次遍历 `_clientObservers` 来调用对应的hook函数