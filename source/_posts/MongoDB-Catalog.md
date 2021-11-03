---
title: MongoDB源码小记 - catalog
date: 2020-01-20 17:36:00
tags:
- MongoDB-OTT
categories:
- MongoDB
- Catalog
---



summary（V1）：src/mongo/db/catalog目录内为对catalog的具体实现，主要通过DatabaseHolder的几个方法对外提供服务。DatabaseHolder被调用的方式有两种：

* catalog_raii.cpp中提供了几个对对应方法的封装
* 部分命令直接调用DatabaseHolder的方法

Collection & Database 是核心的两个类，提供了对db和collection操作的主要入口

CollectionCatalog、ViewCatalog、IndexCatalog是几个目录类


### 接口 - RAII

src/mongo/db/catalog_raii.cpp 文件中存储了RAII风格的类（资源获取即初始化）

* `AutoGetDb`：主要在构造方法DatabaseHolder::get(opCtx)->getDb()获取初始化_db，及其他成员变量初始化
* `AutoGetCollection`：构造方法初始化一个AutoGetDb变量，然后获取对应的collection，及其他成员变量初始化
* `AutoGetOrCreateDb`：与AutoGetDb类似，会判断如果getDb为空，则重新openDb()
* ConcealCollectionCatalogChangesBlock
* ReadSourceScope

### DatabaseHolder

* 定义了几个静态的get、set方法，用于获取一个DataBaseHolder
* 定义了几个抽象的成员方法：
  * getDb()：返回一个opened的database，否则返回nullptr。【IS级别锁】
  * openDb()：返回一个opened的database，或者打开一个未被create、open的database。【x级别锁】
  * dropDb()：物理drop一个database，并将其从metadata中清楚。不会notify复制子系统，也不会做任何一致性check。所以不应该直接被用作用户命令【X级别锁】
  * close()：close指定的database【X级别锁】
  * closeAll()：close所有database【X级别锁】

#### DatabaseHolderImpl是DatabaseHolder的唯一实现

* 成员变量：
  * _m : SimpleMutex互斥锁
  * _dbs : 一个String到Database的Map结构
  * _epoch : 当db reopen的时候，用于分配它一个新的epoch // TODO
* 成员方法：
  * getDb() : 查找_dbs中是否有指定dbName的db，如果有返回对应db，没有则返回null
  * openDb() : 查找_dbs中是否有指定dbName的db，如果有返回对应db。如果没有：
    * 通过CollectionCatalog::getAllCollectionUUIDsFromDb查看collection中是否有记录，没有则打印下创建db的审计日志
    * new 一个DatabaseImpl，然后调用其init后加入到_dbs中
  * dropDb() : 判断是否有collection有index操作，如果没有则将collection删掉，然后调用storageEngine->dropDatabase()
  * dropDatabase大致操作：通过dbName获取collection集合，然后依次遍历获取uuid构建成一个大的recoveryUnit提交到存储引擎
  * close() : _dbs中未找到对应db，则直接返回。找到则
    * 调用CollectionCatalog::removeResource，清理掉CollectionCatalog中记录的对应关系_resourceInformation
    * 调用storageEngine->closeDatabase() //实际未做任何操作，storageEngine中没有db的概念
  * closeAll() : 同 close()方法

### Database

Database是逻辑上database操作的入口，主要对外提供profiling、collection、view的操作入口：

* profiling相关操作：setProfilingLevel、getProfilingLevel、getProfilingNS、setDropPending、isDropPending
* collection相关操作：dropCollection、dropCollectionEvenIfSystem、createCollection、getCollection、getOrCreateCollection、renameCollection、makeUniqueCollectionNamespace
* view相关操作：dropView、createView、getSystemViewsName
* 其他基本操作：init、name、begin、end

#### DatabaseImpl是Database的唯一实现：

* 基本操作：_name记录db名称。_epoch作为db被open的计数。
  * init() : 通过CollectionCatalog获取db下所有的collection，并尝试调用其init方法。通过ViewCatalog获取db下所有view，并调用起reload方法
* `collection`
  * `getCollection`：实际就是调用CollectionCatalog::lookupCollectionByNamespace获取对应Collection
  * `renameCollection` : 从Top中清理fromNss，调用DurableCatalog::renameCollection修改磁盘上的meta信息，调用CollectionCatalog::setCollectionNamespace修改CollectionCatalog中的信息
  * `createCollection` : 
    * 判断是否传入uuid，未传入则自动生成
    * _checkCanCreateCollection检查是否可以create collection，主要是名称长度、是否已存在等判断
    * 通过storageEngine获取DurableCatalog，操作磁盘上的catalog信息
    * 通过Collection::Factory创建一个内存态的collection
    * collection初始化，调用init
    * 注册recoveryUnit的onCommit 事件 setMinimumVisibleSnapshot
    * collection注册到CollectionCatalog中：uuid -> Collection
    * 注册recoveryUnit的onRollback事件（清理在CollectionCatalog中的注册信息）
    * 按需创建id索引
    * 调用getOpObserver的onCreateCollection方法  // TODO
  * `dropCollection`：判断是否是特殊的system库，是的话报错，不是则调用dropCollectionEvenIfSystem
  * `dropCollectionEvenIfSystem`：中间各种判断比较多一些，主要干三件事情
    * _dropCollectionIndexes()：后端就是调用collection->getIndexCatalog()->dropAllIndexes
    * opObserver->onDropCollection()和createCollection相对 // TODO
    * _finishDropCollection()
      * DurableCatalog::dropCollection -- 清理磁盘上的catalog信息
      * CollectionCatalog::deregisterCollection -- 清理内存上的注册信息
      * 调用recoveryUnit()的registerChange // TODO
* `profiling`：_profileName记录profile表的名称。_profile记录是否开启profile
* `view`：_viewsName记录view表的名称

### Collection

Collection是逻辑上collection的入口，主要对外提供document、validate的操作入口

* document：findDoc、deleteDocument、insertDocuments、updateDocument、truncate
* validate：基础setter&getter
* collection属性：isTemporary、isCapped、getCappedCallback、getCappedInsertNotifier
* 状态：dataSize、averageObjectSize、getIndexSize

#### CollectionImpl 是Collection的唯一实现

* 核心属性：`_recordStore` & `_indexCatalog` 记录record&index，`_validator`记录验证器，`_ns`&`_uuid`等记录一下基本属性
* 核心的关于document的CRUD方法主要都是对_recordStore相关操作的封装，增删改等操作可能会涉及到 _indexCatalog及OpObserver的操作

> CollectionInfoCacheImpl // TODO

### IndexCatalog

IndexCatalog和Collection是一一对应关系，表示collection中的index集合，提供了一系列index的操作

#### IndexCatalogImpl

IndexCatalogImpl是IndexCatalog的唯一实现

* 核心属性：_magic用于标记是否完成。_collection用于记录归属的Collection。
      * _unfinishedIndexes用于记录mongo在上次shutdown是未完成的index list，如果不为空则需要调用getAndClearUnfinishedIndexes清理掉
      * _readyIndexes & _buildingIndexes 分别记录ready及building的index

##### 核心方法：init

* init ： 通过DurableCatalog从磁盘中加载归属collection的index，记录unfinished index，其余index会调用_setupInMemoryStructures加载到memory中
* setupInMemoryStructures
  * 根据入参，初始化一个IndexCatalogEntryImpl对象
  * 通过DurableCatalog获取ident，然后基于该值通过storageEngine获取一个SortedDataInterface，最终构建一个IndexAccessMethod对象 // TODO  
  * IndexCatalogEntryImpl初始化（传入IndexAccessMethod对象），并根据isReadyIndex，判断加入_readyIndexes or _buildingIndexes（init传入的均为readyIndex）
  * 根据initFromDisk，判断是否增加recoverUnit的onRollback事件（init传入的为true，不需要增加）

##### 核心方法：插入record

* `indexRecords`：依次遍历readyIndexes & buildingIndexes，调用_indexRecords
* `_indexRecords` ：判断如果是partial index，则使用FilterExpression进行过滤然后调用_indexFilteredRecords
* `_indexFilteredRecords`：调用index->accessMethod()->getKeys 获取一些key的信息。调用_indexKeys
* `_indexKeys`：如果index正在创建中，且创建规则是4.2默认的（即IndexBuildMethod::kHybrid），那么在side table中插入一个key kInsert的操作；否则调用index->accessMethod()->insertKeys插入

##### 核心方法：删除record和更新record与插入基本类似

* unindexRecord -- `_unindexRecord` -- _unindexKeys（如果index正在创建中，且创建规则是4.2默认的（即IndexBuildMethod::kHybrid），那么在side table中插入一个key kDelete的操作；否则调用index->accessMethod()->removeKeys删除）
* updateRecord -- `_updateRecord`（先调用accessMethod的prepareUpdate，然后如果index正在创建中，且创建规则是4.2默认的（即IndexBuildMethod::kHybrid），那么依次调用_unindexKeys和_indexKeys。否则直接调用accessMethod的update方法）

##### 核心方法：删除所有index

* dropAllIndexes：前置判断（是否有building的index等）。调用_dropIndex清理单个index。后置判断是否完全清理
* _dropIndex：
  * 从_readyIndexes&_buildingIndexes中清理index，并在recoveryUnit中注册IndexRemoveChange操作
  * collection的infoCache中droppedIndex
  * 调用_deleteIndexFromDisk，最终调用DurableCatalog->removeIndex在磁盘清理index

##### 核心方法：createIndexOnEmptyCollection

* 前置检查：check collection是否为空。check是否有unfinished index等
* 以kForeground的方式构造一个IndexBuildBlock，并进行init
* 因为空的collection，所以直接调用success
  * entry->accessMethod()->initializeAsEmpty()
  * indexBuildBlock.success() ，


### TODO：

* IndexDescriptor、IndexCatalogEntry、IndexBuildInterceptor、IndexAccessMethod
* 推测：
  * IndexDescriptor是对index的描述
  * IndexCatalogEntry是一个index，包含一个IndexDescriptor
  * IndexAccessMethod应该主要封装的是和storage交互的接口，负责index和磁盘的交互
  * IndexBuildInterceptor应该是4.2新增的用于在index创建过程中适配KHybrid的方式

  * IndexBuildBlock
    * init：
      * 通过_spec获取keyPattern、_indexName、_indexNamespace等信息
      * 调用DurableCatalog->prepareForIndexBuild做一些准备工作
      * IndexCatalogImpl->_setupInMemoryStructures获取一个IndexCatalogEntry对象
        * 如果_method == IndexBuildMethod::kHybrid，那么设置entry的IndexBuildInterceptor
          * 且如果IndexBuildProtocol::kTwoPhase == protocol，通过DurableCatalog->setIndexBuildScanning来这是sideWrite table和constraint table
        * 如果是后台创建索引，在recoveryUnit添加onCommit事件 // TODO
        * 在collection的infoCache中添加index

待读：CollectionCatalog、ViewCatalog、IndexCatalog、CollectionInfoCacheImpl ，index相关的待读
