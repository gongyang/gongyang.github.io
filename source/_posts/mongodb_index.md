---
title: Mongodb 索引详解
date: 2016-5-1
---
使用索引，mongodb可以执行有效率的查询。如果没有索引，mongodb必须对整个集合进行扫描，扫描集合中的每一个文档来找到匹配的查询。如果建立一个合适的索引来进行查询，mongodb可以通过查找索引，可以快速对文档进行定位。

下面我们来看一个例子：
在命令行输入mongo，mongodb会默认连接test数据库。
```
[ggyy@gy-vm03 ~]$ mongo
MongoDB shell version: 3.0.0、
connecting to: test
>show tables
>
```
然后在mongodb shell执行如下代码
```
> for(var i=0;i<100000;i++) {
... db.users.insert({username:'user'+i})
... }
> show collections
users
>
```
现在我们从users集合中查询一条数据
```
> db.users.find({username:"user11223"})
{ "_id" : ObjectId("572027b16dae65d12fb33c33"), "username" : "user11223" }
>
```
成功的找到了数据，但是并看不出什么其他信息。我们在查询语句后面加上[explain](https://docs.mongodb.org/manual/reference/method/cursor.explain/)方法再来看看
```
> db.users.find({username:"user11223"}).explain("executionStats")
{
	"queryPlanner" : {
		"plannerVersion" : 1,
		"namespace" : "test.users",
		"indexFilterSet" : false,
		"parsedQuery" : {
			"username" : {
				"$eq" : "user11223"
			}
		},
		"winningPlan" : {
			"stage" : "COLLSCAN",
			"filter" : {
				"username" : {
					"$eq" : "user11223"
				}
			},
			"direction" : "forward"
		},
		"rejectedPlans" : [ ]
	},
	"executionStats" : {
		"executionSuccess" : true,
		"nReturned" : 1,
		"executionTimeMillis" : 40,
		"totalKeysExamined" : 0,
		"totalDocsExamined" : 100000,
		"executionStages" : {
			"stage" : "COLLSCAN",
			"filter" : {
				"username" : {
					"$eq" : "user11223"
				}
			},
			"nReturned" : 1,
			"executionTimeMillisEstimate" : 30,
			"works" : 100002,
			"advanced" : 1,
			"needTime" : 100000,
			"needFetch" : 0,
			"saveState" : 781,
			"restoreState" : 781,
			"isEOF" : 1,
			"invalidates" : 0,
			"direction" : "forward",
			"docsExamined" : 100000
		}
	},
	"serverInfo" : {
		"host" : "gy-vm03",
		"port" : 27017,
		"version" : "3.0.0",
		"gitVersion" : "a841fd6394365954886924a35076691b4d149168"
	},
	"ok" : 1
}
> 
```
哇，好多内容啊，看不懂没关系，我们只关注"executionTimeMillis","totalDocsExamined"这两个参数，可以发现，集合中的每个文档都被扫描了，并且总时间为40毫秒。如果要是这个集合有几千万的数据，那这个服务器基本上就挂了。
那么，如果我们给username加上一个索引，效果又如何了：
```
> db.users.createIndex({username:1})
{
	"createdCollectionAutomatically" : false,
	"numIndexesBefore" : 1,
	"numIndexesAfter" : 2,
	"ok" : 1
}
> db.users.getIndexes()
[
	{
		"v" : 1,
		"key" : {
			"_id" : 1
		},
		"name" : "_id_",
		"ns" : "test.users"
	},
	{
		"v" : 1,
		"key" : {
			"username" : 1
		},
		"name" : "username_1",
		"ns" : "test.users"
	}
]
>
```
索引被成功的创建，接下来我们再次执行上一条查询语句
```
> db.users.find({username:"user19223"}).explain("executionStats")
{
	"queryPlanner" : {
		"plannerVersion" : 1,
		"namespace" : "test.users",
		"indexFilterSet" : false,
		"parsedQuery" : {
			"username" : {
				"$eq" : "user19223"
			}
		},
		"winningPlan" : {
			"stage" : "FETCH",
			"inputStage" : {
				"stage" : "IXSCAN",
				"keyPattern" : {
					"username" : 1
				},
				"indexName" : "username_1",
				"isMultiKey" : false,
				"direction" : "forward",
				"indexBounds" : {
					"username" : [
						"[\"user19223\", \"user19223\"]"
					]
				}
			}
		},
		"rejectedPlans" : [ ]
	},
	"executionStats" : {
		"executionSuccess" : true,
		"nReturned" : 1,
		"executionTimeMillis" : 0,
		"totalKeysExamined" : 1,
		"totalDocsExamined" : 1,
		"executionStages" : {
			"stage" : "FETCH",
			"nReturned" : 1,
			"executionTimeMillisEstimate" : 0,
			"works" : 2,
			"advanced" : 1,
			"needTime" : 0,
			"needFetch" : 0,
			"saveState" : 0,
			"restoreState" : 0,
			"isEOF" : 1,
			"invalidates" : 0,
			"docsExamined" : 1,
			"alreadyHasObj" : 0,
			"inputStage" : {
				"stage" : "IXSCAN",
				"nReturned" : 1,
				"executionTimeMillisEstimate" : 0,
				"works" : 2,
				"advanced" : 1,
				"needTime" : 0,
				"needFetch" : 0,
				"saveState" : 0,
				"restoreState" : 0,
				"isEOF" : 1,
				"invalidates" : 0,
				"keyPattern" : {
					"username" : 1
				},
				"indexName" : "username_1",
				"isMultiKey" : false,
				"direction" : "forward",
				"indexBounds" : {
					"username" : [
						"[\"user19223\", \"user19223\"]"
					]
				},
				"keysExamined" : 1,
				"dupsTested" : 0,
				"dupsDropped" : 0,
				"seenInvalidated" : 0,
				"matchTested" : 0
			}
		}
	},
	"serverInfo" : {
		"host" : "gy-vm03",
		"port" : 27017,
		"version" : "3.0.0",
		"gitVersion" : "a841fd6394365954886924a35076691b4d149168"
	},
	"ok" : 1
}
> 
```
我们可以看到通过索引，只扫描了一条数据，并且这次查询几乎是没有花时间。
当然使用索引是也是有代价的：对于添加的每一条索引，每次写操作（插入、更新、删除）都将耗费更多的时间。这是因为，当数据发生变化时，不仅要更新文档，还要更新级集合上的所有索引。因此，mongodb限制每个集合最多有64个索引，索引名的长度不能超过125个字符。通常，在一个特定集合中，索引数最好不要超过2个，选择热点键或有可能是性能瓶颈的键做索引。

## 索引类型

### 单键索引
mongodb支持对集合中文档的任一键建立索引（mongodb中的集合就是table，键就是column）。每个集合默认都有一个_id键的索引。用户可以另外添加索引来支持重要的查询和操作。

**创建一个索引**
在users集合上面创建一个单键索引
```
> { "_id" : objectid(...),
>   "name": "bobo",
>   "age" : 20
> }
```
使用**createIndex**命令在name键上面建立索引
```
> db.users.cerateIndex({"name" : 1})
```
**_id键上的索引**
mongodb在我们创建集合的时候，自动会在_id键上面创建_id索引,并且是唯一索引。我们不能从_id键上面移除这个索引。

**内嵌文档键的索引（Indexes on Embedded Fields）**
people collection
```
> {"_id": ObjectId(...),
>  "name": "John Doe",
>  "address": {
>        "street": "Main",
>        "zipcode": "53511",
>        "state": "WI"
>        }
> }
```
对address.zipcode创建索引
```
> db.people.createIndex( { "address.zipcode": 1 } )
```
对如下集合metro创建索引
```
> {
>  _id: ObjectId(...),
>  metro: {
>           city: "New York",
>           state: "NY"
>         },
>  name: "Giant Factory"
> }
```
metro键是一个内嵌文档，包含city，state两个内嵌键，下面的命令可以对metro键创建索引
```
> db.factories.createIndex( { metro: 1 } )
```
如下查询可以利用到基于metro键的索引
```
> db.factories.find( { metro: { city: "New York", state: "NY" } } )
```
但是如下查询却不能使用metro索引
```
> db.factories.find( { metro: { state: "NY", city: "New York" } } )
```
这是因为，我们的查询匹配键，必须和内嵌文档键顺序完全匹配。

### 复合索引
什么是复合索引，就是索引中可以包含集合中文档的多个键。复合索引可以支持要求匹配多个键的查询
例如集合club_posts
```
>{ 
 "_id" : 51, 
 "post_id" : 7353, 
 "sticky" : false, 
 "club_id" : 1 
}
```
应用需要查询post_id键，有时还需要同时查询post_id和club_id键，那么我们可以创建一个复合索引键
```
>db.club_posts.createIndex({post_id:1,club_id:1})
{
	"createdCollectionAutomatically" : false,
	"numIndexesBefore" : 1,
	"numIndexesAfter" : 2,
	"ok" : 1
}
```
**tips:**
我们不能对hashed键创建复合建索，这样会报错。

### 多键索引
假如有文档posts，其结构如下
```
> 
{
	"_id" : 16,
	"body" : "dGagrshrshre",
	"created_at" : ISODate("2013-09-30T06:45:33.862Z"),
	"deleted_at" : null,
	"envious_count" : 1,
	"feedbacks" : [
		{
			"type" : "envious",
			"user_id" : 24,
			"updated_at" : ISODate("2013-09-30T07:01:46.897Z"),
			"created_at" : ISODate("2013-09-30T07:01:46.897Z"),
			"_id" : 37
		}
	],
	"hateful_count" : 0,
	"jealous_count" : 0,
	"type" : "",
	"updated_at" : ISODate("2014-05-04T10:26:40.235Z")
	｝
```
如果我们对feedbacks这个键创建了索引，由于这是一个数组，为了索引一个储存数组的键，MongoDB对数组中的每个元素都添加索引项，我们不需要显示地指定索引为多键类型
创建多键索引
```
>db.posts.createIndex({feedbacks:1})
```
**使用限制**
我们可以创建一个包含有多键索引的复合索引，但是在复合索引中最多只能有一个多键索引。例如，有一个索引 { a: 1, b: 1 }, 以下的文档是允许的
```
>{ _id: 1, a: [1, 2], b: 1, category: "A array" }
{ _id: 2, a: 1, b: [1, 2], category: "B array" }
```
但是，如下的文档就是不被允许的，而且MongoDB将不会把这样的一个文档存储到集合中
```
>{ _id: 1, a: [ 1, 2 ], b: [ 1, 2 ], category: "AB - both arrays" }
```
mongodb还支持地理空间索引和查询、文本索引、哈希索引等。

## 索引属性
### TTL索引
TTL索引是一种特殊索引，通过这种索引MongoDB会过一段时间后自动移除集合中的文档。这对于某些类型的信息来说是一个很理想的特性，例如机器生成的事件数据，日志，会话信息等，这些数据都只需要在数据库中保存有限时间。
使用TTL属性，只需要在创建索引的时候加上expireAfterSeconds方法

例如，如下操作在 log_events 集合的createdAt字段创建了一个索引并指定 expireAfterSeconds的值为60以使过期时间为createdAt指定的时间之后的一分钟
```
> db.log_events.createIndex( { "createdAt": 1 }, { expireAfterSeconds: 60 } )
```
当向 log_events 集合添加文档时，设置 createdAt 字段为当前时间：
```
> db.log_events.insert( {
...    "createdAt": new Date(),
...    "logEvent": 2,
...    "logMessage": "Success!"
... } )
WriteResult({ "nInserted" : 1 })
```
这样log_events集合里面的文档的createdAt字段的值晚于 expireAfterSeconds中指定的秒数时，MongoDB会自动从log_events 集合删除该文档。

### 唯一索引
唯一索引可以让MongoDB拒绝保存那些被索引键的值已经重复的文档。
创建一个唯一索引
```
> db.collection.createIndex( { a: 1 }, { unique: true } )
```
也可以对 复合索引 施加唯一性限制，如下：
```
> db.collection.createIndex( { a: 1, b: 1 }, { unique: true } )
```
这个索引会强制要求 复合 键值的唯一性，而 不是 每个键的唯一性。

### 稀疏索引
稀疏索引中值存储那些有被索引键的文档的索引项，即使被索引键的值是null也会被索引(译者注：请注意，这里对null的处理和那些特殊索引的默认稀疏特性有细微差别，比如文本索引，2d索引等)。索引会跳过所有不包含被索引键的文档。这个索引之所以称为 “稀疏” 是因为它并不包括集合中的所有文档。与之相反，非稀疏的索引会索引每一篇文档，如果一篇文档不含被索引键则为它存储一个null值。
创建一个稀疏索引
```
> db.collection.createIndex( { a: 1 }, { sparse: true } )
```
在A集合上创建稀疏索引
假设集合 scores 有如下文档：
```
>{ "_id" : ObjectId("523b6e32fb408eea0eec2647"), "userid" : "newbie" }
{ "_id" : ObjectId("523b6e61fb408eea0eec2648"), "userid" : "abby", "score" : 82 }
{ "_id" : ObjectId("523b6e6ffb408eea0eec2649"), "userid" : "nina", "score" : 90 }
```
集合在 score 键上有一个稀疏索引：
```
>db.scores.createIndex( { score: 1 } , { sparse: true } )
```
那么，在 scores 集合上执行如下查询，将会利用稀疏索引来返回包含了 score 键且值小于 ($lt) 90 的文档：
```
>db.scores.find( { score: { $lt: 90 } } )
```
由于userid为 "newbie" 的文档不包含 score 键，因此无法满足查询条件，那么查询可以利用稀疏索引来返回如下结果：
```
>{ "_id" : ObjectId("523b6e61fb408eea0eec2648"), "userid" : "abby", "score" : 82 }
```
在A集合上的稀疏索引不会返回完整结果

假设集合 scores 有如下文档：
```
>{ "_id" : ObjectId("523b6e32fb408eea0eec2647"), "userid" : "newbie" }
{ "_id" : ObjectId("523b6e61fb408eea0eec2648"), "userid" : "abby", "score" : 82 }
{ "_id" : ObjectId("523b6e6ffb408eea0eec2649"), "userid" : "nina", "score" : 90 }
```
集合在 score 键上有一个稀疏索引：
```
> db.scores.createIndex( { score: 1 } , { sparse: true } )
```
由于userid为 "newbie" 的文档不包含 score 键， 因此稀疏索引中不包含该文档的索引项。

假设有如下查询，返回 scores 集合中 所有 文档并按照 score 键排序：
```
> db.scores.find().sort( { score: -1 } )
```
即使是按照被索引键排序，MongoDB仍然 不会 选择稀疏索引来匹配这个查询，这是为了可以得到完整的结果集：
```
> {"_id" : ObjectId("523b6e6ffb408eea0eec2649"), "userid" : "nina", "score" : 90 }
{ "_id" : ObjectId("523b6e61fb408eea0eec2648"), "userid" : "abby", "score" : 82 }
{ "_id" : ObjectId("523b6e32fb408eea0eec2647"), "userid" : "newbie" }
```
如果希望使用稀疏索引，请在 hint() 显示指定该索引：
```
>db.scores.find().sort( { score: -1 } ).hint( { score: 1 } )
```
稀疏索引的使用导致了只有那些包含 score 键的文档被返回了：
```
> { "_id" : ObjectId("523b6e6ffb408eea0eec2649"), "userid" : "nina", "score" : 90 }
{ "_id" : ObjectId("523b6e61fb408eea0eec2648"), "userid" : "abby", "score" : 82 }
```

