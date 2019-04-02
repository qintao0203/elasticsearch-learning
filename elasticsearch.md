## 1.索引

​	**索引（名词）**：一个 *索引* 类似于传统关系数据库中的一个 *数据库* ，是一个存储关系型文档的地方。 *索引* (*index*) 的复数词为 *indices* 或 *indexes* 。indices相当于数据库，type相当于表。

​	**索引（动词）**：*索引一个文档* 就是存储一个文档到一个 *索引* （名词）中以便它可以被检索和查询到。这非常类似于 SQL 语句中的 `INSERT` 关键词，除了文档已存在时新文档会替换旧文档情况之外。

​	**倒排索引**：关系型数据库通过增加一个 *索引* 比如一个 B树（B-tree）索引 到指定的列上，以便提升数据检索速度。Elasticsearch 和 Lucene 使用了一个叫做 *倒排索引* 的结构来达到相同的目的。

​	索引实际上是指向一个或者多个物理 *分片* 的 *逻辑命名空间* 。

## 2.CRUD

	### 1).PUT

用来更新以及添加数据

```
PUT /megacorp/employee/1
{
    "first_name" :  "Jane",
    "last_name" :   "Smith",
    "age" :         32,
    "about" :       "I like to collect rock albums",
    "interests":  [ "music" ]
}
```

### 2）GET

检索文档

```
GET /megacorp/employee/1
result：
{
  "_index" :   "megacorp",
  "_type" :    "employee",
  "_id" :      "1",
  "_version" : 1,
  "found" :    true,
  "_source" :  {
      "first_name" :  "John",
      "last_name" :   "Smith",
      "age" :         25,
      "about" :       "I love to go rock climbing",
      "interests":  [ "sports", "music" ]
  }
}

//不会返回元数据
GET /website/blog/123/_source

//批量获取数据
GET /_mget
{
   "docs" : [
      {
         "_index" : "website",
         "_type" :  "blog",
         "_id" :    2
      },
      {
         "_index" : "website",
         "_type" :  "pageviews",
         "_id" :    1,
         "_source": "views"
      }
   ]
}
```

### 3)DELETE

删除文档

```
DELETE /megacorp/employee/1
```

### 4)HEAD

检查文档或者索引是否存在

```
HEAD /megacorp      查看索引是否存在
HEAD /megacorp/employee/1  检查对象是否存在
```

### 5)POST

​	POST与PUT相似都是添加数据，而不同是PUT添加数据需要完整的INDEX/TYPE/ID,POST可以省略ID,es会自动生成唯一id（自动生成的 ID 是 URL-safe、 基于 Base64 编码且长度为20个字符的 GUID 字符串。 这些 GUID 字符串由可修改的 FlakeID 模式生成，这种模式允许多个节点并行生成唯一 ID ，且互相之间的冲突概率几乎为零。）

### 6)PUT

创建查询，指定op_type，如果创建新文档的请求成功执行，Elasticsearch 会返回元数据和一个 `201 Created` 的 HTTP 响应码，如果具有相同的 _index 、 _type 和 _id 的文档已经存在，Elasticsearch 将会返回 409 Conflict 响应码

另一方面，如果具有相同的 `_index` 、 `_type` 和 `_id` 的文档已经存在，Elasticsearch 将会返回 `409 Conflict` 响应码

```js
PUT /website/blog/123?op_type=create
{ ... }
或
PUT /website/blog/123/_create
{ ... }

```

### 7)bulk

请求体：

```js
{ action: { metadata }}\n
{ request body        }\n
{ action: { metadata }}\n
{ request body        }\n
```

示例：

```js
POST /_bulk
{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }} #删除操作不需要请求体
{ "create": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "title":    "My first blog post" }
{ "index":  { "_index": "website", "_type": "blog" }}
{ "title":    "My second blog post" }
{ "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" : 3} }
{ "doc" : {"title" : "My updated blog post"} }\n #最后一个换行符一定要带上

返回结果：
#每个子请求都是独立执行，因此某个子请求的失败不会对其他子请求的成功与否造成影响。 如果其中任何子请求失败，最顶层的 error 标志被设置为 true ，并且在相应的请求报告出错误明细
{
   "took": 4,
   "errors": false, 
   "items": [
      {  "delete": {
            "_index":   "website",
            "_type":    "blog",
            "_id":      "123",
            "_version": 2,
            "status":   200,
            "found":    true
      }},
      {  "create": {
            "_index":   "website",
            "_type":    "blog",
            "_id":      "123",
            "_version": 3,
            "status":   201
      }},
      {  "create": {
            "_index":   "website",
            "_type":    "blog",
            "_id":      "EiwfApScQiiy7TIKFxRCTw",
            "_version": 1,
            "status":   201
      }},
      {  "update": {
            "_index":   "website",
            "_type":    "blog",
            "_id":      "123",
            "_version": 4,
            "status":   200
      }}
   ]
}
```

性能:

*最佳点* ：通过批量索引典型文档，并不断增加批量大小进行尝试。 当性能开始下降，那么你的批量大小就太大了。一个好的办法是开始时将 1,000 到 5,000 个文档作为一个批次, 如果你的文档非常大，那么就减少批量的文档个数。

密切关注你的批量请求的物理大小往往非常有用，一千个 1KB 的文档是完全不同于一千个 1MB 文档所占的物理大小。 一个好的批量大小在开始处理后所占用的物理大小约为 5-15 MB。

## 3.简单搜索

### 1)参数查询

```
GET /megacorp/employee/_search
result:
{
   "took":      6,
   "timed_out": false,
   "_shards": { ... },
   "hits": {
      "total":      2,
      "max_score":  1,
      "hits": [
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "3",
            "_score":         1,
            "_source": {
               "first_name":  "Douglas",
               "last_name":   "Fir",
               "age":         35,
               "about":       "I like to build cabinets",
               "interests": [ "forestry" ]
            }
         },
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "2",
            "_score":         1,
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}

查询某个字段
GET /megacorp/employee/_search?q=last_name:Smith
result:
{
   ...
   "hits": {
      "total":      2,
      "max_score":  0.30685282,
      "hits": [
         {
            ...
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            ...
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

### 2）表达式 -领域特定语言（DSL）

```
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```

####使用过滤器查询

```
GET /megacorp/employee/_search
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
```

####全文搜索

```
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}

result:
{
   ...
   "hits": {
      "total":      2,
      "max_score":  0.16273327,
      "hits": [
         {
            ...
            "_score":         0.16273327, 相关性得分
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            ...
            "_score":         0.016878016, 
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}

```

​	Elasticsearch 默认按照相关性得分排序，即每个文档跟查询的匹配程度。第一个最高得分的结果很明显：John Smith 的 `about` 属性清楚地写着 “rock climbing” 。但为什么 Jane Smith 也作为结果返回了呢？原因是她的 `about` 属性里提到了 “rock” 。因为只有 “rock” 而没有 “climbing” ，所以她的相关性得分低于 John 的。

#### 短语搜索

​	仅匹配同时包含 “rock” *和* “climbing”

```
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
result:
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         }
      ]
   }
}
```

#### 高亮搜索

​	增加一个新的 `highlight` 参数

```
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}

result:
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            },
            "highlight": {
               "about": [
                  "I love to go <em>rock</em> <em>climbing</em>" 
               ]
            }
         }
      ]
   }
}
```

#### 分析-聚合（aggregations）

​	聚合与 SQL 中的 `GROUP BY` 类似但更强大。

```
GET /megacorp/employee/_search
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}
result:
{
   ...
   "hits": { ... },
   "aggregations": {
      "all_interests": {
         "buckets": [
            {
               "key":       "music",
               "doc_count": 2
            },
            {
               "key":       "forestry",
               "doc_count": 1
            },
            {
               "key":       "sports",
               "doc_count": 1
            }
         ]
      }
   }
}
```

​	叫 Smith 的雇员中最受欢迎的兴趣爱好

​	**all_interests**聚合已经变为只包含匹配查询的文档

```
GET /megacorp/employee/_search
{
  "query": {
    "match": {
      "last_name": "smith"
    }
  },
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests"
      }
    }
  }
}
```

聚合还支持分级汇总 。比如，查询特定兴趣爱好员工的平均年龄

```
GET /megacorp/employee/_search
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : { "field" : "age" }
                }
            }
        }
    }
}
result:
  "all_interests": {
     "buckets": [
        {
           "key": "music",
           "doc_count": 2,
           "avg_age": {
              "value": 28.5
           }
        },
        {
           "key": "forestry",
           "doc_count": 1,
           "avg_age": {
              "value": 35
           }
        },
        {
           "key": "sports",
           "doc_count": 1,
           "avg_age": {
              "value": 25
           }
        }
     ]
  }
```

## 4.分片

​	一个分片是一个 Lucene 的实例，以及它本身就是一个完整的搜索引擎。 我们的文档被存储和索引到分片内，但是应用程序是直接与索引而不是与分片进行交互。

​	Elasticsearch 是利用分片将数据分发到集群内各处的。分片是数据的容器，文档保存在分片内，分片又被分配到集群内的各个节点里。 当你的集群规模扩大或者缩小时， Elasticsearch 会自动的在各节点中迁移分片，使得数据仍然均匀分布在集群里。

​	一个分片可以是 *主* 分片或者 *副本* 分片。 索引内任意一个文档都归属于一个主分片，所以主分片的数目决定着索引能够保存的最大数据量。一个主分片最大能够存储 Integer.MAX_VALUE - 128 个文档。

​	一个副本分片只是一个主分片的拷贝。 副本分片作为硬件故障时保护数据不丢失的冗余备份，并为搜索和返回文档等读操作提供服务。在索引建立的时候就已经确定了主分片数，但是副本分片数可以随时修改。默认情况下索引分配五个分片一个副本。

## 5文档

​	通常情况下，*对象* 和 *文档* 是可以互相替换的。有一个区别： 一个对象仅仅是类似于 hash 、 hashmap 、字典或者关联数组的 JSON 对象，对象中也可以嵌套其他的对象。 对象可能包含了另外一些对象。在 Elasticsearch 中， *文档* 有着特定的含义。它是指最顶层或者根对象, 这个根对象被序列化成 JSON 并存储到 Elasticsearch 中，指定了唯一 ID。

### 文档元数据

​	一个文档不仅仅包含它的数据 ，也包含 *元数据* —— *有关* 文档的信息。 三个必须的元数据元素如下：

```
_index：文档在哪存放
```

```
_type：文档表示的对象类别
```

```
_id：文档唯一标识
```

### 冲突处理

​	乐观锁：

​		es使用version控制并发修改，每次执行操作版本都会递增，如果使用就数据覆盖新数据就会出现409错误

​	悲观锁

### 文档存储

​	当我们创建文档时会按照以下公式进行分片分配

```
shard = hash(routing) % number_of_primary_shards
```

​	`	 routing` 是一个可变值，默认是文档的 `_id` ，也可以设置成一个自定义的值。 `routing` 通过 hash 函数生成一个数字，然后这个数字再除以 `number_of_primary_shards` （主分片的数量）后得到 **余数** 。这个分布在 `0` 到 `number_of_primary_shards-1` 之间的余数，就是我们所寻求的文档所在分片的位置。

​	所有的文档 API（ `get` 、 `index` 、 `delete` 、 `bulk` 、 `update` 以及 `mget` ）都接受一个叫做 `routing` 的路由参数 ，通过这个参数我们可以自定义文档到分片的映射。一个自定义的路由参数可以用来确保所有相关的文档——例如所有属于同一个用户的文档——都被存储到同一个分片中。

###新建、索引和删除单个文档

![](G:\学习\笔记\elasticsearch\images\elas_0402.png)

1. 客户端向 `Node 1` 发送新建、索引或者删除请求。
2. 节点使用文档的 `_id` 确定文档属于分片 0 。请求会被转发到 `Node 3`，因为分片 0 的主分片目前被分配在 `Node 3` 上。
3. `Node 3` 在主分片上面执行请求。如果成功了，它将请求并行转发到 `Node 1` 和 `Node 2` 的副本分片上。一旦所有的副本分片都报告成功, `Node 3` 将向协调节点报告成功，协调节点向客户端报告成功。



**consistency**

​	consistency，即一致性。在默认设置下，即使仅仅是在试图执行一个_写_操作之前，主分片都会要求必须要有 _规定数量(quorum)_（或者换种说法，也即必须要有大多数）的分片副本处于活跃可用状态，才会去执行_写_操作(其中分片副本可以是主分片或者副本分片)。这是为了避免在发生网络分区故障（network partition）的时候进行_写_操作，进而导致数据不一致。_规定数量_即：

```
int( (primary + number_of_replicas) / 2 ) + 1
```

​	`consistency` 参数的值可以设为 `one` （只要主分片状态 ok 就允许执行_写_操作）,`all`（必须要主分片和所有副本分片的状态没问题才允许执行_写_操作）, 或 `quorum` 。默认值为 `quorum` , 即大多数的分片副本状态没问题就允许执行_写_操作。

​	注意，*规定数量* 的计算公式中 `number_of_replicas` 指的是在索引设置中的设定副本分片数，而不是指当前处理活动状态的副本分片数。如果你的索引设置中指定了当前索引拥有三个副本分片，那规定数量的计算结果即：

```
int( (primary + 3 replicas) / 2 ) + 1 = 3
```

​	如果此时你只启动两个节点，那么处于活跃状态的分片副本数量就达不到规定数量，也因此您将无法索引和删除任何文档。

### 获取文档

1、客户端向 `Node 1` 发送获取请求。

2、节点使用文档的 `_id` 来确定文档属于分片 `0` 。分片 `0` 的副本分片存在于所有的三个节点上。 在这种情况下，它将请求转发到 `Node 2` 。

3、`Node 2` 将文档返回给 `Node 1` ，然后将文档返回给客户端。

![](G:\学习\笔记\elasticsearch\images\elas_0403.png)

### 局部更新文档

![](G:\学习\笔记\elasticsearch\images\elas_0404.png)

1. 客户端向 `Node 1` 发送更新请求。
2. 它将请求转发到主分片所在的 `Node 3` 。
3. `Node 3` 从主分片检索文档，修改 `_source` 字段中的 JSON ，并且尝试重新索引主分片的文档。 如果文档已经被另一个进程修改，它会重试步骤 3 ，超过 `retry_on_conflict` 次后放弃。
4. 如果 `Node 3` 成功地更新文档，它将新版本的文档并行转发到 `Node 1` 和 `Node 2` 上的副本分片，重新建立索引。 一旦所有副本分片都返回成功， `Node 3` 向协调节点也返回成功，协调节点向客户端返回成功。

###  为什么是有趣的格式？

​	当我们早些时候在[代价较小的批量操作](https://www.elastic.co/guide/cn/elasticsearch/guide/current/bulk.html)章节了解批量请求时， 您可能会问自己， "为什么 `bulk` API 需要有换行符的有趣格式，而不是发送包装在 JSON 数组中的请求，例如 `mget` API？" 。

​	为了回答这一点，我们需要解释一点背景：在批量请求中引用的每个文档可能属于不同的主分片， 每个文档可能被分配给集群中的任何节点。这意味着批量请求 `bulk` 中的每个 *操作* 都需要被转发到正确节点上的正确分片。

如果单个请求被包装在 JSON 数组中，那就意味着我们需要执行以下操作：

- 将 JSON 解析为数组（包括文档数据，可以非常大）
- 查看每个请求以确定应该去哪个分片
- 为每个分片创建一个请求数组
- 将这些数组序列化为内部传输格式
- 将请求发送到每个分片

这是可行的，但需要大量的 RAM 来存储原本相同的数据的副本，并将创建更多的数据结构，Java虚拟机（JVM）将不得不花费时间进行垃圾回收。

相反，Elasticsearch可以直接读取被网络缓冲区接收的原始数据。 它使用换行符字符来识别和解析小的 `action/metadata` 行来决定哪个分片应该处理每个请求。

这些原始请求会被直接转发到正确的分片。没有冗余的数据复制，没有浪费的数据结构。整个请求尽可能在最小的内存中处理。

## 6 脚本

​	脚本模块使您可以使用脚本来评估自定义表达式。例如，您可以使用脚本将“脚本字段”作为搜索请求的一部分返回，或者为查询评估自定义分数。

默认的脚本语言是[`Painless`](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/modules-scripting-painless.html)。其他`lang`插件使您可以运行用其他语言编写的脚本。在任何地方都可以使用脚本，您可以包含一个`lang`参数来指定脚本的语言。

![/images/微信截图_20190327173623.png](G:\学习\笔记\elasticsearch\images\微信截图_20190327173623.png)



## 7搜索

```js
GET /_search
result:
{
   "hits" : {
      "total" :       14, #匹配到的文档总数
      "hits" : [
        {
          "_index":   "us",
          "_type":    "tweet",
          "_id":      "7",
          "_score":   1,#衡量了文档与查询的匹配程度。默认情况下，首先返回最相关的文档结果，就是说，返回的文档是按照 _score 降序排列的
          "_source": {
             "date":    "2014-09-17",
             "name":    "John Smith",
             "tweet":   "The Query DSL is really powerful and flexible",
             "user_id": 2
          }
       },
        ... 9 RESULTS REMOVED ...
      ],
      "max_score" :   1
   },
   "took" :           4,#执行整个搜索请求耗费了多少毫秒
   "_shards" : {
      "failed" :      0,
      "successful" :  10,
      "total" :       10
   },
   "timed_out" :      false
}
```

简单示例：

```
/_search
在所有的索引中搜索所有的类型
/gb/_search
在 gb 索引中搜索所有的类型
/gb,us/_search
在 gb 和 us 索引中搜索所有的文档
/g*,u*/_search
在任何以 g 或者 u 开头的索引中搜索所有的类型
/gb/user/_search
在 gb 索引中搜索 user 类型
/gb,us/user,tweet/_search
在 gb 和 us 索引中搜索 user 和 tweet 类型
/_all/user,tweet/_search
在所有的索引中搜索 user 和 tweet 类型
```

### 分页

Elasticsearch 接受 `from` 和 `size` 参数：

- `size`

  显示应该返回的结果数量，默认是 `10`

- `from`

  显示应该跳过的初始结果数量，默认是 `0`

如果每页展示 5 条结果，可以用下面方式请求得到 1 到 3 页的结果：

```js
GET /_search?size=5
GET /_search?size=5&from=5
GET /_search?size=5&from=10
```

**深度分页：**

​	理解为什么深度分页是有问题的，我们可以假设在一个有 5 个主分片的索引中搜索。 当我们请求结果的第一页（结果从 1 到 10 ），每一个分片产生前 10 的结果，并且返回给 *协调节点* ，协调节点对 50 个结果排序得到全部结果的前 10 个。

​	现在假设我们请求第 1000 页--结果从 10001 到 10010 。所有都以相同的方式工作除了每个分片不得不产生前10010个结果以外。 然后协调节点对全部 50050 个结果排序最后丢弃掉这些结果中的 50040 个结果。

​	可以看到，在分布式系统中，对结果排序的成本随分页的深度成指数上升。这就是 web 搜索引擎对任何查询都不要返回超过 1000 个结果的原因。



###映射（Mapping）

​	描述数据在每个字段内如何存储

```js
GET /gb/_mapping/tweet
查询索引字段类型：
{
   "gb": {
      "mappings": {
         "tweet": {
            "properties": {
               "date": {
                  "type": "date",
                  "format": "strict_date_optional_time||epoch_millis"
               },
               "name": {
                  "type": "string"
               },
               "tweet": {
                  "type": "string"
               },
               "user_id": {
                  "type": "long"
               }
            }
         }
      }
   }
}
```

###分析（Analysis）

​	全文是如何处理使之可以被搜索的

###领域特定查询语言（Query DSL）

​	Elasticsearch 中强大灵活的查询语言

```js
GET /_search
{
    "query": YOUR_QUERY_HERE
}
GET /_search
{
    "query": {
        "match_all": {}
    }
}

结构：
{
    QUERY_NAME: {
        FIELD_NAME: {
            ARGUMENT: VALUE,
            ARGUMENT: VALUE,...
        }
    }
}
    
示例：
GET /_search
{
    "query": {
        "match": {
            "tweet": "elasticsearch"
        }
    }
}
```

####组合查询

```js
{
    "bool": {
        "must":     { "match": { "tweet": "elasticsearch" }},
        "must_not": { "match": { "name":  "mary" }},
        "should":   { "match": { "tweet": "full text" }},
        "filter":   { "range": { "age" : { "gt" : 30 }} }
    }
}
```

#####match_all 查询

`match_all` 查询简单的 匹配所有文档。在没有指定查询方式时，它是默认的查询：

```js
{ "match_all": {}}
```

##### multi_match 查询

`multi_match` 查询可以在多个字段上执行相同的 `match` 查询：

```js
{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}
```

##### range 查询

`range` 查询找出那些落在指定区间内的数字或者时间：

```js
{
    "range": {
        "age": {
            "gte":  20,
            "lt":   30
        }
    }
}
```

被允许的操作符如下：

- `gt`

  大于

- `gte`

  大于等于

- `lt`

  小于

- `lte`

  小于等于

##### term 查询

`term` 查询被用于精确值 匹配，这些精确值可能是数字、时间、布尔或者那些 `not_analyzed` 的字符串：

```js
{ "term": { "age":    26           }}
{ "term": { "date":   "2014-09-01" }}
{ "term": { "public": true         }}
{ "term": { "tag":    "full_text"  }}
```

`term` 查询对于输入的文本不 *分析* ，所以它将给定的值进行精确查询。

#####  terms 查询

`terms` 查询和 `term` 查询一样，但它允许你指定多值进行匹配。如果这个字段包含了指定值中的任何一个值，那么这个文档满足条件：

```js
{ "terms": { "tag": [ "search", "full_text", "nosql" ] }}
```

和 `term` 查询一样，`terms` 查询对于输入的文本不分析。它查询那些精确匹配的值（包括在大小写、重音、空格等方面的差异）。

##### exists 查询和 missing 查询

`exists` 查询和 `missing` 查询被用于查找那些指定字段中有值 (`exists`) 或无值 (`missing`) 的文档。这与SQL中的 `IS_NULL` (`missing`) 和 `NOT IS_NULL` (`exists`) 在本质上具有共性：

```js
{
    "exists":   {
        "field":    "title"
    }
}
```

这些查询经常用于某个字段有值的情况和某个字段缺值的情况。

#####Bool查询器

`bool` 查询来实现你的需求。这种查询将多查询组合在一起，成为用户自己想要的布尔查询。它接收以下参数：

- `must`

  文档 *必须* 匹配这些条件才能被包含进来。

- `must_not`

  文档 *必须不* 匹配这些条件才能被包含进来。

- `should`

  如果满足这些语句中的任意语句，将增加 `_score` ，否则，无任何影响。它们主要用于修正每个文档的相关性得分。

- `filter`

  *必须* 匹配，但它以不评分、过滤模式来进行。这些语句对评分没有贡献，只是根据过滤标准来排除或包含文档。

  

```

带有过滤时间查询
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "range": { "date": { "gte": "2014-01-01" }} 
        }
    }
}

混合bool查询
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }}
        ],
        "filter": {
          "bool": { 
              "must": [
                  { "range": { "date": { "gte": "2014-01-01" }}},
                  { "range": { "price": { "lte": 29.99 }}}
              ],
              "must_not": [
                  { "term": { "category": "ebooks" }}
              ]
          }
        }
    }
}
```

##### constant_score 查询

尽管没有 `bool` 查询使用这么频繁，`constant_score` 查询也是你工具箱里有用的查询工具。它将一个不变的常量评分应用于所有匹配的文档。它被经常用于你只需要执行一个 filter 而没有其它查询（例如，评分查询）的情况下。

可以使用它来取代只有 filter 语句的 `bool` 查询。在性能上是完全相同的，但对于提高查询简洁性和清晰度有很大帮助。

```js
{
    "constant_score":   {
        "filter": {
            "term": { "category": "ebooks" } 
        }
    }
}
```

#####使用步骤技巧：

1.根据需求写sql进行翻译

2.将sql翻译成es查询进行匹配

3.将es语句使用Java api进行组装

```
搜索帖子bool查询器
{
	"from": 0,
	"size": 20,
	"query": {
		"bool": {
			"must": [{
				"bool": {
					"should": [{
						"match": {
							"postText": {
								"query": "审核",
								"operator": "OR",
								"analyzer": "ik_smart",
								"prefix_length": 0,
								"max_expansions": 50,
								"fuzzy_transpositions": true,
								"lenient": false,
								"zero_terms_query": "NONE",
								"auto_generate_synonyms_phrase_query": true,
								"boost": 1.0
							}
						}
					}, {
						"match": {
							"postTitle": {
								"query": "审核",
								"operator": "OR",
								"analyzer": "ik_smart",
								"prefix_length": 0,
								"max_expansions": 50,
								"fuzzy_transpositions": true,
								"lenient": false,
								"zero_terms_query": "NONE",
								"auto_generate_synonyms_phrase_query": true,
								"boost": 1.0
							}
						}
					}],
					"adjust_pure_negative": true,
					"boost": 1.0
				}
			}, {
				"bool": {
					"must": [{
						"term": {
							"auditStatus": {
								"value": 1,
								"boost": 1.0
							}
						}
					}],
					"adjust_pure_negative": true,
					"boost": 1.0
				}
			}],
			"adjust_pure_negative": true,
			"boost": 1.0
		}
	},
	"highlight": {
		"pre_tags": ["<font color=\"#72b1d0\">"],
		"post_tags": ["</font>"],
		"fields": {
			"postText": {
				"fragment_size": 20
			},
			"postTitle": {
				"fragment_size": 20
			}
		}
	}
}
```

#### 排序

默认情况下，返回的结果是按照 *相关性* 进行排序的——最相关的文档排在最前

```js
GET /_search
{
    "query" : {
        "bool" : {
            "filter" : { "term" : { "user_id" : 1 }}
        }
    },
    "sort": { "date": { "order": "desc" }}
}
```

#### 多级排序

假定我们想要结合使用 `date` 和 `_score` 进行查询，并且匹配的结果首先按照日期排序，然后按照相关性排序：

```js
GET /_search
{
    "query" : {
        "bool" : {
            "must":   { "match": { "tweet": "manage text search" }},
            "filter" : { "term" : { "user_id" : 2 }}
        }
    },
    "sort": [
        { "date":   { "order": "desc" }},
        { "_score": { "order": "desc" }}
    ]
}
```

## 8.聚合

类似于 DSL 查询表达式， 聚合也有 *可组合* 的语法：独立单元的功能可以被混合起来提供需要的自定义行为。

两个主要的概念：

- *桶（Buckets）*

  满足特定条件的文档的集合

- *指标（Metrics）*

  对桶内的文档进行统计计算

这就是全部了！每个聚合都是一个或者多个桶和零个或者多个指标的组合。翻译成的SQL语句：

```sql
SELECT COUNT(color) 
FROM table
GROUP BY color 
```

| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/1.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/aggs-high-level.html#CO183-1) | `COUNT(color)` 相当于指标。 |
| ------------------------------------------------------------ | --------------------------- |
| [![img](https://www.elastic.co/guide/cn/elasticsearch/guide/current/images/icons/callouts/2.png)](https://www.elastic.co/guide/cn/elasticsearch/guide/current/aggs-high-level.html#CO183-2) | `GROUP BY color` 相当于桶。 |

桶在概念上类似于 SQL 的分组（GROUP BY），而指标则类似于 `COUNT()` 、 `SUM()` 、 `MAX()` 等统计方法。



###查询

```js
GET /cars/transactions/_search
{
    "size" : 0,#设置查询数据数
    "aggs" : { #聚合操作被置于顶层参数 aggs 之下（完整形式 aggregations 同样有效）
        "popular_colors" : { #自定义名称
            "terms" : { #定义单个桶的类型 terms
              "field" : "color"
            }
        }
    }
}
result:
{
...
   "hits": {
      "hits": [] 
   },
   "aggregations": {
      "popular_colors": { 
         "buckets": [
            {
               "key": "red", 
               "doc_count": 4 
            },
            {
               "key": "blue",
               "doc_count": 2
            },
            {
               "key": "green",
               "doc_count": 2
            }
         ]
      }
   }
}
```

###添加度量指标

```js
#每种颜色汽车的平均价格是多少
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": { 
            "avg_price": { 
               "avg": {
                  "field": "price" 
               }
            }
         }
      }
   }
}
result:
{
...
   "aggregations": {
      "colors": {
         "buckets": [
            {
               "key": "red",
               "doc_count": 4,
               "avg_price": { 
                  "value": 32500
               }
            },
            {
               "key": "blue",
               "doc_count": 2,
               "avg_price": {
                  "value": 20000
               }
            },
            {
               "key": "green",
               "doc_count": 2,
               "avg_price": {
                  "value": 21000
               }
            }
         ]
      }
   }
...
}
```

### 嵌套桶

```js
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": {
            "avg_price": { 
               "avg": {
                  "field": "price"
               }
            },
            "make": { 
                "terms": {
                    "field": "make" 
                }
            }
         }
      }
   }
}
result:
{
...
   "aggregations": {
      "colors": {
         "buckets": [
            {
               "key": "red",
               "doc_count": 4,
               "make": { 
                  "buckets": [
                     {
                        "key": "honda", 
                        "doc_count": 3
                     },
                     {
                        "key": "bmw",
                        "doc_count": 1
                     }
                  ]
               },
               "avg_price": {
                  "value": 32500 
               }
            },

...
}
```

- 红色车有四辆。
- 红色车的平均售价是 $32，500 美元。
- 其中三辆是 Honda 本田制造，一辆是 BMW 宝马制造。

```js
#计算最低最高价格
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs": {
      "colors": {
         "terms": {
            "field": "color"
         },
         "aggs": {
            "avg_price": { "avg": { "field": "price" }
            },
            "make" : {
                "terms" : {
                    "field" : "make"
                },
                "aggs" : { 
                    "min_price" : { "min": { "field": "price"} }, 
                    "max_price" : { "max": { "field": "price"} } 
                }
            }
         }
      }
   }
}
result:
{
...
   "aggregations": {
      "colors": {
         "buckets": [
            {
               "key": "red",
               "doc_count": 4,
               "make": {
                  "buckets": [
                     {
                        "key": "honda",
                        "doc_count": 3,
                        "min_price": {
                           "value": 10000 
                        },
                        "max_price": {
                           "value": 20000 
                        }
                     },
                     {
                        "key": "bmw",
                        "doc_count": 1,
                        "min_price": {
                           "value": 80000
                        },
                        "max_price": {
                           "value": 80000
                        }
                     }
                  ]
               },
               "avg_price": {
                  "value": 32500
               }
            },
...
```

- 有四辆红色车。
- 红色车的平均售价是 $32，500 美元。
- 其中三辆红色车是 Honda 本田制造，一辆是 BMW 宝马制造。
- 最便宜的红色本田售价为 $10，000 美元。
- 最贵的红色本田售价为 $20，000 美元。

### 按时间统计

```js
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs": {
      "sales": {
         "date_histogram": {
            "field": "sold",
            "interval": "month", #时间间隔要求是日历术语 (如每个 bucket 1 个月)
            "format": "yyyy-MM-dd" #日期格式以便 buckets 的键值便于阅读
         }
      }
   }
}

result:
{
   ...
   "aggregations": {
      "sales": {
         "buckets": [
            {
               "key_as_string": "2014-01-01",
               "key": 1388534400000,
               "doc_count": 1
            },
            {
               "key_as_string": "2014-02-01",
               "key": 1391212800000,
               "doc_count": 1
            },
            {
               "key_as_string": "2014-05-01",
               "key": 1398902400000,
               "doc_count": 1
            },
            {
               "key_as_string": "2014-07-01",
               "key": 1404172800000,
               "doc_count": 1
            },
            {
               "key_as_string": "2014-08-01",
               "key": 1406851200000,
               "doc_count": 1
            },
            {
               "key_as_string": "2014-10-01",
               "key": 1412121600000,
               "doc_count": 1
            },
            {
               "key_as_string": "2014-11-01",
               "key": 1414800000000,
               "doc_count": 2
            }
         ]
...
}
```

空buckets

```js
GET /cars/transactions/_search
{
   "size" : 0,
   "aggs": {
      "sales": {
         "date_histogram": {
            "field": "sold",
            "interval": "month",
            "format": "yyyy-MM-dd",
            "min_doc_count" : 0, #强制返回空 buckets
            "extended_bounds" : { #强制返回整年
                "min" : "2014-01-01",
                "max" : "2014-12-31"
            }
         }
      }
   }
}
```