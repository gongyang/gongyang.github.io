---
title: mongodb 在复制集上创建修改索引
date: 2016-3-6
---
在讲mongodb如何在复制集上创建或修改索引的操作步骤前，我们先来了解一下mongodb在创建索引时会产生什么影响。
mongodb在默认情况下，创建索引会阻塞数据库中所有其他操作。当在一个集合上创建索引时，存储了这个集合的数据库变成不可读不可写的状态直到索引建立完毕。任何需要所有数据库中读或者写锁的操作(例如 listDatabases )将会等待，直到索引创建完成。
对于可能需要长时间运行的索引创建操作，可以考虑 background 选项，这样MongoDB数据库在索引创建期间仍然是可用的。
例如，在 people 集合的 zipcode 键上创建一个索引，这个过程在后台运行，可以使用如下方式：
```
db.people.createIndex( { zipcode: 1}, {background: true} )
```
默认地，MongoDB索引创建的 background 是 false 。索引在创建过程中,查询将不会使用部分建立的索引：索引只有在建立完毕之后才是可用的。

**tips：**
使用后台索引的方式，虽然可以保证数据库可用，但是会影响mongodb的性能，后台索引创建操作使用的是一种增量的方式，这会比普通的 “前台” 创建过程慢。如果索引大于现有的内存，那么这个增量处理过程将会比前台创建过程 久得多 。
如果MongoDB在后台创建一个索引，您不能执行其他会涉及到该集合的管理操作，包括 repairDatabase 命令，删除集合(例如 db.collection.drop() )，和 compact 命令。当后台创建索引时，这些操作会返回一个错误。
If your application includes createIndex() operations, and an index doesn’t exist for other operational concerns, building the index can have a severe impact on the performance of the database.
为了避免性能问题，请确保您的应用在启动时就使用 getIndexes() 方法或者其他 equivalent method for your driver 检查索引是否存在，如果不存在请退出应用。或者在生产实例上使用不同的应用代码在指定的维护时间窗口期间创建索引。

处于以上考虑，对于mongodb复制集的索引创建和修改，mongodb官方建议使用下面步骤

**停止一个从节点**
停止一个从节点上的 mongod 进程。重启 mongod 进程， 不要 带 --replSet 选项且指定一个不同的端口。限制这个实例是运行在 “standalone” 模式下的。

**创建索引**
在 mongo shell 里通过 ensureIndex() 方法创建新索引（mongodb3.2版本，“ensureIndex() ”方法已经停用，使用createIndex()方法来创建索引 ）
例如，为了在 records 集合的 username 键上创建一个递增索引，可以使用如下 mongo shell 操作：
```
db.records.createIndex( { username: 1 } )
```

**重启 mongod 进程**
当索引创建完毕，在原有端口上用选项 --replSet 重启 mongod 实例，让其加入到复制集中，同步数据。

**在所有从节点上创建索引**
对于复制集中的每个从节点，按如下步骤创建索引：

停止一个从节点
创建索引
重启 mongod 进程

**主节点**
使用 mongo shell 中的 rs.stepDown() 方法让主节点下台(step down)，这样主节点平滑地过渡为从节点，且允许复制集选举其他成员为主节点。
然后重启上述索引创建步骤，在(原)主节点上创建索引:
停止一个从节点
创建索引
重启 mongod 进程
在后台创建索引会比前台方式耗时更久，且会生成不够紧凑的索引 结构。此外，后台创建索引可能会影响主节点的写性能。但是，在后台建立索引允许复制集在MongoDB建立索引期间持续写操作。
