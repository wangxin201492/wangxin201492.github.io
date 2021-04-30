---
title: MongoDB serverStatus实现原理
date: 2020-04-21 13:15:09
tags:
- MongoDB-OTT
- MongoDB-serverStatus
categories:
- MongoDB
- command
- serverStatus
---

## Category

[TOC]

 `CmdServerStatus` 的 run() 函数是 `serverStatus` 命令的具体实现

- 首先将一些全局信息添加到result中
- 遍历`_sections`，调用`appendSection`，将所有注册的`Section`结果添加到result中
- 调用`MetricTree::theMetricTree->appendTo()`，将所有`Metric`指标添加到result中
- 其他一些信息的补充

 

同时 `CmdServerStatus` 还维护了一个 `_sections` （map结构）存储section名称和具体 `ServerStatusSection` 的对应关系。提供 `addSection` 函数将制定section添加到 `_sections` 中



## Section指标

`ServerStatusSection` 是所有 `section` 指标的基类，构造方法调用 `CmdServerStatusInstantiator::getInstance()->addSection()` 将自身注册到 `CmdServerStatus` 中。并声明 virtual 函数 `includeByDefault` , `addRequiredPrivileges` , `generateSection` , `appendSection` 由子类定义实现，子类主要需要按需求定义`includeByDefault` , `generateSection` 的具体实现。

- `includeByDefault` 用于说明是否默认被包含
- `generateSection`&`appendSection`   一般实现其中一个即可，父类方法中`appendSection` 调用了 `generateSection` ，所以一般只需要实现 `generateSection` 即可。

 

`CmdServerStatusInstantiator` 是包装 `CmdServerStatus` 用于获取 `CmdServerStatus` 单例对象的一个struct，提供 `getInstance()` 完成单例对象获取。所以上面讲到的 `ServerStatusSection` 构造时调用 `CmdServerStatusInstantiator::getInstance()->addSection()` ，最终完成了自身到 `CmdServerStatus` 的 _sections 注册。

 

serverStatus命令运行时将遍历自身的 _sections ，依次根据 `includeByDefault` 配置，调用其 appendSection 完成 response BSON 的构建。

 

> 其他 Command 都是在类声明的时候同时定义了一个对象，完成到 CommandMap 的注册。而 `CmdServerStatus` 由于需要 `CmdServerStatusInstantiator` 构建单例对象，所以是在 `CmdServerStatusInstantiator` 声明时定义了一个对象，完成 `CmdServerStatus` 到 CommandMap 的注册行为。

 

## Metric指标

`MetricTree`维护了一个树状结构(下文简称Tree) 用于存储所有的 metric指标 及其 path 对应关系，同时提供一个静态变量 `theMetricTree` 对外提供服务。同时提供 `add` 和 `appendTo` 函数分别用于将 Tree 添加到 `theMetricTree` 以及将 Tree 的数据以BSON的形式输出。

- `add` 函数用于将 `ServerStatusMetric` 添加到 `Tree` : `Tree` 维护 `metrics path` 和 `ServerStatusMetric` 的对应关系，内部是由 2 个 map 构成的，如果当前 `metrics path` 是叶子节点（即 path 中不存在"."），那么存储到``path --> ServerStatusMetric` 的 map 中，反之取出 path 的第一级路径，存储到 第一级路径 --> `MetricTree` （不存在则新加一个） ，然后递归调用完成添加

- - 如果 path 是以 "." 为开头，则添加到 Tree 的顶级路径，否则会在提供的 path 前添加 "metrics." 前缀（即添加到metrics的子节点上）

- `appendTo` 函数用于将 `Tree` 的数据以BSON的形式输出 ： 对于非叶子节点，会递归调用 `appendTo` 函数。对于叶子节点则会调用 `ServerStatusMetric` 的 `appendAtLeaf` 函数。最终根据所有 `ServerStatusMetric` 的 path 产生对应的的BSON

 

`ServerStatusMetric` 是所有 metric指标 的基类，主要提供构造方法调用 `MetricTree::theMetricTree->add(this)` 将自身注册到 `MetricTree` 中。并声明一个 `virtual appendAtLeaf` 函数由子类定义实现。该类有4个子类：

- 模板类 `ServerStatusMetricField` 是一个被广泛应用的子类。增加一个T，用于实现大多数需求：都是 k-v 的结构， `appendAtLeaf`将 **k & v** 直接append到bson中

- - 举例场景：所有command的一个成员变量，用于做类似命令与请求次数的绑定，便于serverStatus拿到结果

- `MemBase (= .mem.bits )` ，获取MongoDB使用的内存信息

- `ClusterCursorStats (= cursor )`，cursor 信息

- `ReplExecutorSSM (= repl.executor )`，// TODO

 

serverStatus命令运行时，将调用 `theMetricTree` 的 `appendTo` 函数，完成 response BSON 的构建

 

## Section指标 & Metric指标 实现对比

ServerStatusMetric 和 ServerStatusSection 下文统称为Impl

|                                                 | Section指标                                                  | Metric指标                                                   |
| ----------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 都有一个单例的对象管理                          | CmdServerStatusInstantiator::getInstance()                   | MetricTree::theMetricTree                                    |
| 都是 name -- Impl  的对应关系                   | CmdServerStatus  中_sections是一个map结构  name -- ServerStatusSection | ServerStatusMetric中基于2个map结构实现了一个树状结构  叶子节点也都是  name -- ServerStatusMetric |
| Impl都提供一个函数将结果  append 到 response 中 | generateSection&appendSection                                | appendTo                                                     |

 

区别应该有2点：

1. `ServerStatusSection`的append函数 `generateSection`&`appendSection` 包含了 opCtx 和 configElement 。定义分别是
   * `virtual BSONObj generateSection(OperationContext* opCtx, const BSONElement&      configElement) const;``
   * ``virtual void appendSection(OperationContext* opCtx, const BSONElement&      configElement, BSONObjBuilder* result) const;`
2. `ServerStatusSection`提供了 `includeByDefault` 和 `addRequiredPrivileges` 对输出结构的控制力更强一些

 

看起来是对于serverStatus默认添加且是计数行为的使用 `ServerStatusMetric` 比较方便简单一些。其他情况尤其对于 opCtx 有依赖或者需要控制对serverStatus输出的还是使用 `ServerStatusSection` 操控力更强一些。