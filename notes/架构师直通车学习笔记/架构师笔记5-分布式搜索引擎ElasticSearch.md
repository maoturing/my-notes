# 0. 学习计划

- 简介与安装
- 基本操作
- DSL 搜索
- 分页与查询
- 可视化插件，IK分词器，词库
- 集群与整合 SpringBoot 实现搜索功能
- geo 坐标搜索
- Logstatsh 数据同步
- 整合电商项目



# 1. 简介与安装

目前的搜索功能就是简单的查询数据库，查询关键字 “好吃” 的 SQL 如下所示

```sql
SELECT
	i.id AS itemId,
	i.item_name AS itemName,
	i.sell_counts AS sellCounts
FROM
	items i
WHERE
	i.item_name LIKE '%好吃%'
ORDER BY
	i.item_name ASC
```



当前搜索功能的缺点

- 不支持空格
- 分词查询
- 搜索内容不能高亮显示
- 海量数据查询



## 1.1 ES 核心术语

- 索引 index：对应数据库中的表
- 类型 type：（已过期）表的分类，类似于用户表、登录表、权限表都可以归结为 user type
- 文档 document：一行数据，存储一行数据是以 json 形式
- 字段 fields：列，属性

```json
student_index
{id:111, name: bob, ahe 18},
{id:222, name: tom, ahe 20},
{id:333, name: jack, ahe 22},
```

- 映射 mapping：表结构的定义，

- 近实时 NRT：Near real time，新的document 被创建后，到用户能够搜索到的延迟时间

- 节点 node：每一个服务器节点

- shard：数据分片

- replica：数据备份

  ![image-20220126173024946](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220126173024946.png)





## 1.2 倒排索引

![image-20220126174057016](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220126174057016.png)

![image-20220126173406405](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220126173406405.png)



根据关键字去查询内容，如果是左侧的表，需要一个一个记录去必对，检查文档内容是否包含关键字。而右侧的倒排索引表，可以使用索引，快速找到与关键字匹配的记录，然后获得包含该关键字的文档 id，这样对于搜索引擎来说，查询效率更高。

而且右侧的倒排索引表，包含了词频TF属性，即关键字在该记录中出现的次数，可以利用该属性来对搜索结果进行排序，方便搜索引擎排序。

倒排索引起源于实际应用中，需要根据属性的值来查找记录。这种索引表的每一项，都包括一个属性值和包含该属性值的各个记录地址。由于不是根据记录来确定属性，而是根据属性来确定记录的位置，所以称之为**倒排索引**。



## 1.3 安装 ElasticSearch

- docker 安装 ElasticSearch

```cmd
docker pull docker.io/elasticsearch:7.1.1
docker run -d --name es -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" b0e9f9f047e6
```

```
-d：后台启动
--name：容器名称
-p：端口映射
-e：设置环境变量
discovery.type=single-node：单机运行
b0e9f9f047e6：镜像id
如果启动不了，可以加大内存设置：-e ES_JAVA_OPTS="-Xms512m -Xmx512m"
```

访问`http://localhost:9200/`测试是否启动成功。



- Linux 安装 ElasticSearch

  详细安装步骤、配置、启动关闭等操作，见1-9文字章节



- 安装 ElasticSearch header 插件

  在chrome 市场搜索 ElasticSearch header 插件并安装，打开该插件，可以使用 web 界面操作 ElasticSearch 





官方文档可以看 [Elasticsearch: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)



# 2. 基本操作

## 2.1 使用 ElasticSearch Head 插件操作

打开 ElasticSearch Head 插件，连接 ElasticSearch 服务`http://localhost:9200/`，然后就可以看到节点信息，在索引Tab也可以创建索引

![image-20220126210009970](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220126210009970.png)



## 2.2 使用 postman 操作

- 查看集群状态 `http://localhost:9200/_cluster/health`，返回结果如下

  ```json
  {
      "cluster_name": "docker-cluster",
      "status": "green",
      "timed_out": false,
      "number_of_nodes": 1,
      "number_of_data_nodes": 1,
      "active_primary_shards": 8,
      "active_shards": 8,
      "relocating_shards": 0,
      "initializing_shards": 0,
      "unassigned_shards": 0,
      "delayed_unassigned_shards": 0,
      "number_of_pending_tasks": 0,
      "number_of_in_flight_fetch": 0,
      "task_max_waiting_in_queue_millis": 0,
      "active_shards_percent_as_number": 100.0
  }
  ```

  可以看到，集群名称为 docker-cluster，集群状态为 green，集群节点数为 1，集群活跃的分片为 8 等信息

  

- 查看所有索引状态  `localhost:9200/_cat/indices?v`，返回结果如下

```
health status index      uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   index_demo O29csTO9TAW3EmwrxTkyXg   3   0          0            0       849b           849b
green  open   index_temp nDVZ6k5EQcOM3Age0ti9-w   5   0          0            0      1.1kb          1.1kb
```

### 1. 操作索引 index

- 获取`index_demo`索引信息  `GET  http://localhost:9200/index_demo`，返回结果如下

  ```json
  {
  	"index_demo": {
  		"aliases": {},
  		"mappings": {},
  		"settings": {
  			"index": {
  				"creation_date": "1643198328384",
  				"number_of_shards": "5",
  				"number_of_replicas": "0",
  				"uuid": "yesHqXT-S8uyxX-9Y1SP4w",
  				"version": {
  					"created": "7040299"
  				},
  				"provided_name": "index_demo"
  			}
  		}
  	}
  }
  ```

  

- 删除`index_demo`索引 `DELETE  http://localhost:9200/index_demo`，返回结果如下

  ```
  {
  	"acknowledged": true
  }
  ```

- 创建`index_demo`索引，携带的索引配置信息如下 

  `PUT  http://localhost:9200/index_demo`

  ```json
  // Body-raw-json
  {
  	"settings": {
  		"index": {
  			"number_of_shards": "3",
  			"number_of_replicas": "0"
  		}
  	}
  }
  ```

  返回结果如下

  ```json
  {
      "acknowledged": true,
      "shards_acknowledged": true,
      "index": "index_demo"
  }
  ```

### 2. 操作映射 mapping

- 设置映射 mapping，前面说了，创建索引相当于创建表，映射 mapping 相当于表结构。现在发送请求创建索引并设置映射

  `PUT http://localhost:9200/index_product`

  ```json
  // Body-raw-json
  {
      "mappings": {
          "properties": {
              "productname": {
                  "type": "text",
                  "index": "true"
              },
              "company": {
                  "type": "keyword",
                  "index": "true"                
              },
              "price": {
                  "type": "text",
                  "index": "false"                
              }
          }
      }
  }
  ```

  

  返回结果如下

  ```json
  {
      "acknowledged": true,
      "shards_acknowledged": true,
      "index": "index_product"
  }
  ```

  mappings 属性和创建索引 index时发送的 settings 属性属于同一级，二者可以合并，相当于创建索引的同时设置映射  

  `PUT http://localhost:9200/index_product2`

  ```json
  // Body-raw-json
  {
  	"settings": {
  		"index": {
  			"number_of_shards": "3",
  			"number_of_replicas": "0"
  		}
  	},
      "mappings": {
          "properties": {
              "productname": {
                  "type": "text",
                  "index": "true"
              },
              "company": {
                  "type": "keyword",
                  "index": "true"                
              },
              "price": {
                  "type": "text",
                  "index": "false"                
              }
          }
      }
  }
  ```

- 文本分析，向指定索引发送指定字段的文本信息，获取分词结果 

   `GET  http://localhost:9200/index_product/_analyze`

  ```json
  // Body-raw-json
  {
      "field": "productname",
      "text": "xiaomi phone high performance"
  }
  ```

  返回的分词结果如下，可以看到根据空格将文本分成了多个关键字，测试了中文，发现会将中文分为一个个汉字，而不是分词

  ```json
  {
      "tokens": [
          {
              "token": "xiaomi",
              "start_offset": 0,
              "end_offset": 6,
              "type": "<ALPHANUM>",
              "position": 0
          },
          {
              "token": "phone",
              "start_offset": 7,
              "end_offset": 12,
              "type": "<ALPHANUM>",
              "position": 1
          },
          {
              "token": "high",
              "start_offset": 13,
              "end_offset": 17,
              "type": "<ALPHANUM>",
              "position": 2
          },
          {
              "token": "performance",
              "start_offset": 18,
              "end_offset": 29,
              "type": "<ALPHANUM>",
              "position": 3
          }
      ]
  }
  ```

  然后将请求携带的内容修改为

  ```json
  {
      "field": "company",
      "text": "xiaomi phone high performance"
  }
  ```

  返回结果如下，由于`company`字段是`keyword`类型，所以不会进行分词

  ```json
  {
      "tokens": [
          {
              "token": "xiaomi phone high performance",
              "start_offset": 0,
              "end_offset": 29,
              "type": "word",
              "position": 0
          }
      ]
  }
  ```

- 修改映射，只能添加字段，无法修改字段，为索引`index_product`添加两个字段

  `POST  http://localhost:9200/index_product/_mapping`

  ```json
  {
      "properties": {
          "size": {
              "type": "integer"  // 不设置index默认为true
          },
          "id": {
              "type": "long",
              "index": "false"                
          }
      }
  }
  ```

  

**主要数据类型**

支持的数据类型有`integer`, `long`, `float`, `double`, `byte`, `short`, `boolean`, `date`, `object`, `keyword`, `text`

**注意:** 属性一旦被创建，就不能修改了，但是可以新增额外属性

**字符串**

ElasticSearch 新版本将 String 类型划分为了`text`和 `keyword`类型，

- `text`：支持分词和倒排索引的文本内容，如商品名称，商品详情，商品介绍
- `keyword`：精确匹配，不支持分词和倒排索引，直接匹配搜索，如手机号，品牌名称，订单状态

### 3. 操作文档 doc

- 新增文档，首先在浏览器创建索引`my_doc`，然后发送请求新增文档（一行数据），下面的参数`1`表示隐藏主键`_id`为`1`，若`_id`为`1`的行已有文档，则会被覆盖

  `POST  http://localhost:9200/my_doc/_doc/1`

  ```json
  {
      "id": 1001,
      "name": "imooc-1",
      "desc": "imooc is very good1",
      "create_date": "2020-12-20"
  }
  ```

  返回结果如下，在插入数据时会自动生成属性字段。通过第三行数据也可以看出，数据类型不是强约束，date 类型也能写入普通字符串。`_id`的值与连接中的 `1` 一致，**如果链接中省略这个`1`，会自动生成**。

  ```json
  {
      "_index": "my_doc",
      "_type": "_doc",
      "_id": "1",
      "_version": 1,
      "result": "created",  // 如果当前行存在数据, 则显示'updated'
      "_shards": {
          "total": 1,
          "successful": 1,
          "failed": 0
      },
      "_seq_no": 6,
      "_primary_term": 2
  }
  ```

![image-20220128002645664](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220128002645664.png)

- 删除文档，在请求链接中指定要删除的文档`_id`为`3`

  `DELETE  http://localhost:9200/my_doc/_doc/3`

  ```json
  {
      "_index": "my_doc",
      "_type": "_doc",
      "_id": "3",
      "_version": 2,     // 每操作一次, 版本号加1
      "result": "deleted",
      "_shards": {
          "total": 1,
          "successful": 1,
          "failed": 0
      },
      "_seq_no": 9,
      "_primary_term": 2
  }
  ```

- 更新文档，更新指定`_id`行的指定字段`name`属性的值。如果需要全量修改替换，参考新建文档

  `POST  http://localhost:9200/my_doc/_doc/1/_update`

  ```json
  {
      "doc": {
          "name": "慕课网"
      }
  }
  ```

  返回结果

  ```json
  {
      "_index": "my_doc",
      "_type": "_doc",
      "_id": "1",
      "_version": 4,
      "result": "updated",
      "_shards": {
          "total": 1,
          "successful": 1,
          "failed": 0
      },
      "_seq_no": 17,
      "_primary_term": 2
  }
  ```

- 查询文档，根据`_id`查询指定行数据

  `GET  http://localhost:9200/my_doc/_doc/1`

  返回结果如下

  ```json
  {
      "_index": "my_doc",
      "_type": "_doc",
      "_id": "1",
      "_version": 4,
      "_seq_no": 17,
      "_primary_term": 2,
      "found": true,
      "_source": {
          "id": 1001,
          "name": "慕课网",
          "desc": "imooc is very good1",
          "create_date": "2020-12-20"
      }
  }
  ```

- 查询文档，定制结果集，查询指定行的指定属性值，这里只查询`_id=1`的`name, create_date` 属性值

  `GET  http://localhost:9200/my_doc/_doc/1?_source=name,create_date`

  ```json
  {
      "_index": "my_doc",
      "_type": "_doc",
      "_id": "1",
      "_version": 4,
      "_seq_no": 17,
      "_primary_term": 2,
      "found": true,
      "_source": {
          "name": "慕课网",
          "create_date": "2020-12-20"
      }
  }
  ```

  同样的，查询指定列的所有数据

  `GET  http://localhost:9200/my_doc/_doc/_search?_source=name,create_date`

  

- 查询全部文档，即所有数据

  ` GET  http://localhost:9200/my_doc/_search`

```json
{
    "took": 1,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 3,
            "relation": "eq"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "my_doc",
                "_type": "_doc",
                "_id": "2",
                "_score": 1.0,
                "_source": {
                    "id": 1002,
                    "name": "imooc-2",
                    "desc": "imooc is very good2",
                    "create_date": "2022-1-13"
                }
            },
            {
                "_index": "my_doc",
                "_type": "_doc",
                "_id": "3",
                "_score": 1.0,
                "_source": {
                    "id": 1003,
                    "name": "imooc-3",
                    "desc": "imooc is very good3",
                    "create_date": "aaa"
                }
            },
            {
                "_index": "my_doc",
                "_type": "_doc",
                "_id": "1",
                "_score": 1.0,
                "_source": {
                    "id": 1001,
                    "name": "慕课网",
                    "desc": "imooc is very good1",
                    "create_date": "2020-12-20"
                }
            }
        ]
    }
}
```



**查询结果中的元数据**

- _index：文档数据所属的索引，类似表名
- type：文档数据的类型，新版本为`_doc`
- _id：文档数据的唯一标识，类似主键，可以自动生成，也可以手动指定
- _score：查询相关度，是否与搜索条件匹配
- _version：版本号
- _source：文档数据，表中的数据，是json格式



- 判断文档是否存在

  `HEAD  http://localhost:9200/my_doc/_doc/1`

  若存在该文档，则返回`200`，否则返回`404`



> HEAD 请求，与GET请求相同，只不过服务器响应时不会返回消息体  body。
>
> 一个HEAD请求的响应中，HTTP头中包含的元信息应该和一个GET请求的响应消息相同。这种方法可以用来获取请求中隐含的元信息，而不用传输实体本身。也经常用来测试超链接的有效性、可用性和最近的修改。
>
> 由于这种 HTTP 请求只需要返回 head 而不用传递 body，那么传输效率相比 GET 请求更高。



- 乐观锁更新文档数据，首先查询`_id=1`的数据，查询结果如下

  ```json
  {
      "_index": "my_doc",
      "_type": "_doc",
      "_id": "1",
      "_version": 4,
      "_seq_no": 17,
      "_primary_term": 2,
      "found": true,
      "_source": {
          "id": 1001,
          "name": "慕课网",
          "desc": "imooc is very good1",
          "create_date": "2020-12-20"
      }
  }
  ```

  可以看到该文档的`_seq_no=17`，`_primary_term=2`，在使用并发情况下，借助乐观锁修改文档数据，发送请求

  `POST  http://localhost:9200/my_doc/_doc/1?if_seq_no=17&if_primary_term=2`

  ```json
  {
      "doc": {
          "name": "MOOC网"
      }
  }
  ```

  返回结果如下，可以看到文档的`_seq_no=18`，`_primary_term=2`

  ```json
  {
      "_index": "my_doc",
      "_type": "_doc",
      "_id": "1",
      "_version": 5,
      "result": "updated",
      "_shards": {
          "total": 1,
          "successful": 1,
          "failed": 0
      },
      "_seq_no": 18,
      "_primary_term": 2
  }
  ```

  如果再次发送该请求，会因为版本冲突更新失败，需要`_seq_no=17`，实际上`_seq_no=18`，返回结果如下

  > version conflict, required seqNo [17], primary term [2]. current document has seqNo [18] and primary term [2]

  ```json
  {
      "error": {
          "root_cause": [
              {
                  "type": "version_conflict_engine_exception",
                  "reason": "[1]: version conflict, required seqNo [17], primary term [2]. current document has seqNo [18] and primary term [2]",
                  "index_uuid": "tBp7lvSLRze5BRLbYiWU3w",
                  "shard": "0",
                  "index": "my_doc"
              }
          ],
          "type": "version_conflict_engine_exception",
          "reason": "[1]: version conflict, required seqNo [17], primary term [2]. current document has seqNo [18] and primary term [2]",
          "index_uuid": "tBp7lvSLRze5BRLbYiWU3w",
          "shard": "0",
          "index": "my_doc"
      },
      "status": 409
  }
  ```

  

  > ElasticSearch 新版本使用 _seq_no 和 _primary_term 来做版本控制，替代了 version，在并发情况下，使用这两个属性进行乐观锁更新数据，避免并发问题

  - _seq_no：
  - _primary_term：







# 3. 分词器

## 3.1 内置分词器

对文本分词，发送请求。这里选择自带的`standard`分词器

`POST  http://localhost:9200/_analyze`

```json
{
    "analyzer": "standard",
    "text": "I study in imooc.com"
}
```

返回结果如下，可以看到根据空格将文本分词，并且将大写转换为了小写

```json
{
    "tokens": [
        {
            "token": "i",
            "start_offset": 0,
            "end_offset": 1,
            "type": "<ALPHANUM>",
            "position": 0
        },
        {
            "token": "study",
            "start_offset": 2,
            "end_offset": 7,
            "type": "<ALPHANUM>",
            "position": 1
        },
        {
            "token": "in",
            "start_offset": 8,
            "end_offset": 10,
            "type": "<ALPHANUM>",
            "position": 2
        },
        {
            "token": "imooc.com",
            "start_offset": 11,
            "end_offset": 20,
            "type": "<ALPHANUM>",
            "position": 3
        }
    ]
}
```



内置分词器，缺点是不支持对中文分词

- standard：根据空格进行分词，大写字母会被转为小写
- simple：按照非字母分词，包括根据空格、数字、标点符号进行分词，大写字母会被转为小写
- whitespace：根据空格进行分词，不转换大小写
- stop：取出无意义单词，如 the a an is
- keyword：不进行分词









## 3.2 安装分词器

- Docker 容器中安装

1. 进入 Elasticsearch 容器命令行，`cd bin`进入`bin` 目录

2. 下载 IK 分词器，版本号需要与 elasticsearch  一致，执行命令

   `./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.4.2/elasticsearch-analysis-ik-7.4.2.zip`



- Linux 上安装

1. 下载 IK 分词器，版本号需要与 elasticsearch  一致 https://github.com/medcl/elasticsearch-analysis-ik/releases
2. 在 Elasticsearch 根目录创建文件夹 `cd your-es-root/plugins/ && mkdir ik`
3. 将前面下载的 IK 分词器解压到该目录`your-es-root/plugins/ik`



## 3.3 使用 IK 分词器

IK 分词器有两种`ik_smart`和`ik_max_word`

- `ik_max_word`: 会将文本做最细粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,中华人民,中华,华人,人民共和国,人民,人,民,共和国,共和,和,国国,国歌”，会穷尽各种可能的组合，适合 Term Query；

- `ik_smart`: 会做最粗粒度的拆分，比如会将“中华人民共和国国歌”拆分为“中华人民共和国,国歌”，适合 Phrase 查询。



`POST http://localhost:9200/_analyze`

```json
{
    "analyzer": "ik_smart",
    "text": "中华人民共和国国歌"
}
```

返回结果

```json
{
    "tokens": [
        {
            "token": "中华人民共和国",
            "start_offset": 0,
            "end_offset": 7,
            "type": "CN_WORD",
            "position": 0
        },
        {
            "token": "国歌",
            "start_offset": 7,
            "end_offset": 9,
            "type": "CN_WORD",
            "position": 1
        }
    ]
}
```



## 3.4 扩展中文词库

对下面的文本进行分词，会遇到一个问题，分词器无法识别`慕课网`这个词，会分为三个词：慕、课、网。这说明当前的分词器还是有缺陷，针对特定环境下的分词会出现偏差，比如网络流行语，分词就很容易出现错误。

`POST http://localhost:9200/_analyze`

```json
{
    "analyzer": "ik_smart",
    "text": "小毛在慕课网学习编程"
}
```

```json
{
    "tokens": [
        {
            "token": "慕",
            "start_offset": 3,
            "end_offset": 4,
            "type": "CN_CHAR",
            "position": 2
        },
        {
            "token": "课",
            "start_offset": 4,
            "end_offset": 5,
            "type": "CN_CHAR",
            "position": 3
        },
        {
            "token": "网",
            "start_offset": 5,
            "end_offset": 6,
            "type": "CN_CHAR",
            "position": 4
        }
    ]
}
```



1. 扩展中文词库

docker 环境下，进入命令行，修改配置

```cmd
cd elasticsearch/config/analysis-ik
# 编辑IK分词器配置文件
vi IKAnalyzer.cfg.xml

# 扩展词库配置为 custom.dic
<entry key="ext_dict">custom.dic</entry>

# 在同级目录下创建扩展词库文件
vi custom.dic
# 添加关键词
慕课网
尚硅谷
```

2. 重启容器，elasticsearch 也会自动重启，配置就能生效，然后进行测试，返回结果如下

`POST http://localhost:9200/_analyze`

```json
{
    "analyzer": "ik_smart",
    "text": "小毛在慕课网学习编程"
}
```

```json
{
    "tokens": [
        {
            "token": "慕课网",
            "start_offset": 3,
            "end_offset": 6,
            "type": "CN_WORD",
            "position": 2
        }
    ]
}
```



IK 分词器的配置文件`IKAnalyzer.cfg.xml`中可以扩展词典，热更新和其他详细配置见[官方文档](https://github.com/medcl/elasticsearch-analysis-ik)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict">custom/mydict.dic;custom/single_word_low_freq.dic</entry>
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords">custom/ext_stopword.dic</entry>
 	<!--用户可以在这里配置远程扩展字典 -->
	<entry key="remote_ext_dict">location</entry>
 	<!--用户可以在这里配置远程扩展停止词字典-->
	<entry key="remote_ext_stopwords">http://xxx.com/xxx.dic</entry>
</properties>
```





# 4. DSL 搜索

## 4.1 数据准备

1. 创建`shop`索引

2. 建议映射，`username` 属性类型是`keyword`，表示不分词，即如果要查询`username`，必须全等。

   `POST  http://localhost:9200/shop/_mapping`

   ```json
   {
   	// 与2.2.1建立映射的json不同, 因为其请求连接中没有指明mapping,
       // 所以需要在json中指明
       "properties": {
           "id": {
               "type": "long"
           },
           "age": {
               "type": "integer"
           },
           "username": {
               "type": "keyword"
           },
           "nickname": {
               "type": "text",
               "analyzer": "ik_max_word"
           },
           "money": {
               "type": "float"
           },
           "desc": {
               "type": "text",
               "analyzer": "ik_max_word"
           },
           "sex": {
               "type": "byte"
           },
           "birthday": {
               "type": "date"
           },
           "face": {
               "type": "text",
               "index": false
           }
       }
   }
   ```

3. 添加数据，参考文档3-2

   `POST  http://localhost:9200/shop/_doc/1002`

   ```json
   {
       "id": "1002",
       "age": "19",
       "username": "jay jay",
       "nickname": "周杰伦",
       "money": "77.8",
       "desc": "今天上下班都很堵，车流量很大",
       "sex": "1",
       "birthday": "1994-12-21",
       "face": "http://www.imooc.com/static/img/22222"
   }
   ```



## 4.2 搜索(查询)

- 查询，`_search?q=desc:aaa&q=age:20`，`q`表示查询，`:`来分割属性和属性值

`GET  http://localhost:9200/shop/_search?q=desc:慕课网`

返回结果如下，共查询到两条记录`total.value=2`。

```json
{
    "took": 6,
    "timed_out": false,
    "_shards": {
        "total": 3,
        "successful": 3,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 0.81734043,
        "hits": [
            {
                "_index": "shop",
                "_type": "_doc",
                "_id": "1001",
                "_score": 0.81734043,
                "_source": {
                    "id": "1001",
                    "age": "18",
                    "username": "imoocAmazing",
                    "nickname": "慕课网",
                    "money": "88.8",
                    "desc": "我在慕课网学习java和前端,学习到了很多知识",
                    "sex": "0",
                    "birthday": "1992-12-21",
                    "face": "http://www.imooc.com/static/img/1123213"
                }
            },
            {
                "_index": "shop",
                "_type": "_doc",
                "_id": "1003",
                "_score": 0.6810339,
                "_source": {
                    "id": "1003",
                    "age": "20",
                    "username": "bigFace",
                    "nickname": "飞翔的巨鹿",
                    "money": "66.8",
                    "desc": "慕课网团队和导游坐飞机去海外旅游,去了新马里",
                    "sex": "1",
                    "birthday": "1996-12-21",
                    "face": "http://www.imooc.com/static/img/333333"
                }
            }
        ]
    }
}
```



- keyword 查询

若有一条数据如下，如果发送请求`_search?q=nickname:super`，可以查询到这条记录；如果发送`_search?q=username:super`，则查询不到这条记录，因为`username`属性类型是`keyword`，不会进行分词，必须全等才能查询到，`_search?q=username:super hero`即可以查询到这条记录。

```json
{
    "username": "super hero",
    "nickname": "super hero",
}
```

- json 查询，一般情况下我们不使用地址栏查询，而是携带 json 进行查询

`POST  http://localhost:9200/shop/_search`

```json
{
    "query": {
        "match": {
            "nickname": "周杰伦"
        }
    }
}
```





- 请求错误 - json 格式错误

```json
{
    "query" {
        "match": {
            "nickname": "周杰伦"
        }
    }
}
```

返回结果如下，可以看到是 json 解析异常，说明大概率是 json 格式有问题，预料之外的字符`{`，希望用冒号 colon 来分隔字段名和值，在第 2 行 第 14 个字符。

检查一下，是我们缺少了冒号 `"query": {`

```json
"error": {
    "type": "json_parse_exception",
    "reason": "Unexpected character ('{' (code 123)): 
    	was expecting a colon to separate field name and value\n 
    	at [Source: org.elasticsearch.transport.netty4.ByteBufStreamInput@28a8f3d9; 
    	line: 2, column: 14]"
}
```

- 请求错误2 - 关键字错误

```json
{
    "query" {
        "match1": {
            "nickname": "周杰伦"
        }
    }
}
```

返回结果如下，可以看到是解析异常，原因是`match1`关键字没有注册，在第 3 行第 19 个字符

```json
"error": {
    "type": "parsing_exception",
    "reason": "no [query] registered for [match1]",
    "line": 3,
    "col": 19
}
```

- 查询拥有某个字段的数据，若数据具有`username`字段，则返回

`POST  http://localhost:9200/shop/_search`

```json
{
    "query": {
        "exists": {
            "field": "username"
        }
    }
}
```

返回结果如下，查询到了全部数据

```json
"total": {
"value": 4,
"relation": "eq"
}
//....
```



- 查询所有数据，类似于`select * from shop`

  `POST    http://localhost:9200/shop/_search`
  
  ```json
  {
      "query": {
          "match_all": {}
      }
  }
  ```
  
- 查询指定字段，类似于`select id, nickname, age from shop  limit 0,10`

  `POST    http://localhost:9200/shop/_search`

  ```json
  {
      "query": {
          "match_all": {}
      },
      "_source": [
          "id",
          "nickname",
          "age"
      ],
      "from": 0,		// 分页, 从第0条开始,显示10条数据
      "size": 10
  }
  ```

- 查询条件不分词，term：术语，术语自然不能分词查询

  `POST http://localhost:9200/shop/_search`

  ```JSON
  {
      "query": {
          "term": {
              "desc": "慕课网"
          }
      }
  }
  ```

  查询`desc`字段包含`慕课网`的数据，对包含慕、课、网的数据则进行过滤。与`match`相对，下面的查询会对查询条件分词进行查询。

  比如`普朗克常量`就是一个术语，如果返回的内容是介绍科学家普朗克生平的，则与期望不符。

  ```json
  {
      "query": {
          "match": {
              "desc": "慕课网"
          }
      }
  }
  ```

  ![image-20220202142639004](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220202142639004.png)



- 多术语查询 - terms，数据中包含任一查询条件，即符合条件并返回

  `POST http://localhost:9200/shop/_search`

  ```JSON
  {
      "query": {
          "terms": {
              "desc": ["慕课网", "编程", "学习"]
          }
      }
  }
  ```

  返回包含`慕课网`或`编程`或`学习`的数据

- 查询多个词

  `POST http://localhost:9200/shop/_search`

  ```JSON
  {
      "query": {
          "match_phrase": {
              "desc": {
                  "query": "万有引力 常量",
                  "slop": 5
                 
              }
          }
      }
  }
  ```

  返回包含`万有引力`和`常量`的数据，并且两个词之间间隔的词数不超过 `5`。

  比如"万有引力定律的常量是…."符合条件，而"万有引力是科学史上一大发现…….普朗克常量...."则不符合条件

- match 扩展 - operator

  - or：搜索内容分词后，只要一个词匹配就符合条件
  - and：搜索内容分词后，所有词都要包含，才符合条件

  `POST http://localhost:9200/shop/_search`

  ```JSON
  {
      "query": {
          "match": {
              "desc": {
                  "query": "编程慕课网",
                  "operator": "or"
              }
          }
      }
  }
  // 与上面的完全相同
  {
      "query": {
          "match": {
              "desc": "编程慕课网",
          }
      }
  }
  ```

  返回包含分词后`编程`或`慕课网`的数据，如果将`or`改为`and`，则两个词都要包含才符合条件。

  除此之外，还可以使用`minumum_should_match`来指定匹配精度为60%，用户查询内容分词后有10个词语，若数据中包含7个词语，则说明匹配度为 70%，大于60%，符合条件。

  ```json
  {
      "query": {
          "match": {
              "desc": {
                  "query": "编程慕课网",
                  "minumum_should_match": "60%"
              }
          }
      }
  }
  ```

- 根据文档主键搜索 - ids

  `POST http://localhost:9200/shop/_search`

  ```JSON
  {
      "query": {
          "ids": {
              "type": "_doc",
              "values": ["1001", "1002", "1003"]
          }
      }
  }
  ```

   返回指定 id 的数据

- 在多个字段中查询 - multi_match

  `POST http://localhost:9200/shop/_search`

```JSON
{
    "query": {
        "multi_match": {
            "query": "慕课网",
            "fields": ["desc", "nickname"]
        }
    }
}
```

字段`desc`或`nickname`中包含查询内容，则符合条件并返回

- 在多个字段中查询 - 权重

  `POST http://localhost:9200/shop/_search`

```JSON
{
    "query": {
        "multi_match": {
            "query": "慕课网",
            "fields": ["desc", "nickname^10"]
        }
    }
}
```

`nickname^10`表示提升 10 倍相关性，也就是说搜索时以`nickname`为准，`desc`为辅。比如搜索商品，商品名称要比商品简介的权重要高。

- 组合多重查询

  - must：查询必须匹配搜索条件
  - should：查询匹配满足一个以上条件
  - must_not：不匹配搜索条件，一个都不要满足

  `POST    http://localhost:9200/shop/_search`

  ```json
  {
      "query": {
          "bool": {
              "must": [
                  {
                      "match": {
                          "desc": "慕"
                      }
                  },
                  {
                      "match": {
                          "nickname": "慕"
                      }
                  }
              ],
              "should": [
                  {
                      "match": {
                          "sex": "0"
                      }
                  }
              ],
              "must_not": [
                  {
                      "term": {
                          "birthday": "1992-12-24"
                      }
                  }
              ]
          }
      }
  }
  ```

返回`desc`和`nickname`都包含`慕`，`sex`为`0`，且`birthday`不为`1992-12-24`的数据。



- 为指定词语加权 - boost

`POST    http://localhost:9200/shop/_search`

```json
{
    "query": {
        "bool": {
            "should": [
                {
                    "match": {
                        "desc": {
                            "query": "手机",
                            "boost": 20
                        }
                    }
                },
                {
                    "match": {
                        "desc": {
                            "query": "移动电话",
                            "boost": 2
                        }
                    }
                }
            ]
        }
    }
}
```



- 过滤器 - post_filter

  - gte 大于等于，lte 小于等于，gt 大于，lt 小于

  实操：查询`desc`包含慕课网的相关数据，过滤出价格`money`大于60且小于1000的数据

  `POST http://localhost:9200/shop/_search`

  ```JSON
  {
      "query": {
          "match": {
              "desc": "慕课网"
          }
      },
      // post_filter用于查询后, 对结果数据的筛选
      "post_filter": {
          "range": {
              "money": {
                  "gt": 60,
                  "lt": 1000
              }
          }
      }
  }
  ```

- 排序 - sort

  `POST http://localhost:9200/shop/_search`

  实操：对搜索结果按年龄`age`升序，年龄相同则按`money`升序

  ```JSON
  {
      "query": {
          "match": {
              "desc": "慕课网"
          }
      },
      // post_filter用于查询后, 对结果数据的筛选
      "post_filter": {
          "range": {
              "money": {
                  "gt": 60,
                  "lt": 1000
              }
          }
      },
      "sort": [
          {
              "age": "asc"
          },
          {
              "money": "asc"
          }
      ]
  }
  ```



- 高亮 - highlight

  `POST http://localhost:9200/shop/_search`

  ```JSON
  {
      "query": {
          "match": {
              "desc": "慕课网"
          }
      },
      "highlight": {
          "range": {
              "pre_tags": ["<tag>"],
              "post_tags": ["</tag>"],
              "fields": {
                  "desc": {}
              }
          }
      }
  }
  ```

默认的高亮标签为`<em></em>`，需要在前端对标签自定义样式。

- 模糊搜索 ，用户在搜索时出现错别字，搜索引擎会自动纠正，然后尝试匹配索引库中的数据



# 5. 分页与批量查询

​         

// 待补充



- 深度分页
- 提升搜索量
- scroll 滚动搜索
- 批量操作 - bulk





# 6. 集群

下图是 ElasticSearch 集群，粗框是主分片，细框是副本分片，分别存储在不同的服务器上。

![image-20220202154028468](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220202154028468.png)



- 集群不仅可以实现高可用，也能实现海量数据存储的横向扩展

- 同一个分片的主与副本不能放在同一个服务器中，因为一旦宕机，这个分片的数据就无法访问了
- 副本分片是主分片的备份，主分片挂了，可以访问副本分片



创建一个索引 `b`，分片数为 3，副本数为 1。

从下图可以看出有三台服务器节点 node，索引`b`有 3 个主分片，3 个备份分片，分别保存在不同的节点上。粗框是主分片，细框是副本分片，星号是主节点。

![image-20220202154739706](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220202154739706.png)



## 6.1 集群搭建

见 5-4 章节



## 6.2 集群脑裂

> 什么是脑裂？

如果旺盛网络中断或服务器宕机，那么集群会有可能被划分为两个部分，各自有自己的 master 来管理，这就是脑裂。





**脑裂解决方案**

master 主节点要经过多个 master 节点共同选举后才能成为新的主节点。就跟班级里选班长一样，并不是一个人决定的，需要半数以上的人同意。

解决原理：半数以上的节点同意选举，节点才可以成为新的 master

- discovery.zen.minimum_master_nodes = (N/2) + 1
- N是集群中的 master 节点的数量，也就是那些`node.master=true`设置的服务器节点数量

在最新的 7.x 版本中，discovery.zen.minimum_master_nodes 这个参数已经被移除了，这一块内容完全由 es 自身去管理，避免了脑裂问题，选举也会非常快。



## 6.3 集群文档读写原理（重点）

### 1. 文档写原理

1. 客户端连接集群，集群会分配其中一个节点作为协调节点（coordinating node）来处理请求，这里的协调节点是 node-2

2. 客户端发送写请求（包括新增文档，删除文档，修改文档），协调节点会使用 hash 规则将其转发到对应的主分片上，这里的对应主分片是 P1，然后进行写操作

3. 当主分片写操作完毕后，将数据同步到对应的副本分片，这里是 R1

4. 待副本分片写操作完毕后，通知协调节点

5. 协调节点将写操作的结果响应给客户端

   ![image-20220202161305675](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220202161305675.png)

## 2. 文档读原理

1. 客户端连接集群，集群会分配其中一个节点作为协调节点（coordinating node）来处理请求，这里的协调节点是 node-1

2. 客户端发送读请求，协调节点会根据文档的 id 进行 hash，获得其存储的分片，这里存储指定数据的主分片是 P0，副本分片是 R0，然后进行读操作。

   读操作会轮询访问主分片或副本分片，来读取数据，起到负载均衡的作用，可见副本分片不只是备份数据。

3. 读取数据完毕后，通知协调节点
4. 协调节点将读取的结果响应给客户端

![image-20220202162410046](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220202162410046.png)



# 7. SpringBoot 整合 ElasticSearch









# 8. Logstash

Logstash 是用来同步数据库数据到 ElasticSearch 的工具







1. 在[官网下载](https://www.elastic.co/cn/downloads/past-releases#logstash)与 ElasticSearch 同版本的 Logstash，有 win 版和 linux 版，这里我们直接选择 win 版，下载后解压即可
2. Logstash 依赖 java 环境，需要安装 jdk
3. 



## ElasticSearch 搜索原理





## Kibana

Kibana 是一个免费且开放的返回



# logstatsh

# 集群











> **小技巧：** 

# 待补充




# 进阶学习



# 推荐阅读

# 参考文档




# 学习记录

2022年1月26日16:55:09
