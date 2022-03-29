# 1. Redis 入门



## 1.1 分布式架构概述

> 什么是分布式架构？

- 不同的业务（功能模块）分散部署在不同的服务器
- 每个子系统负责一个或多个不同的业务功能模块
- 服务之间可以交互与通信，HTTP RPC 等
- 分布式系统设计对用户透明，用户不关心分布式系统的细节
- 每个子系统可以扩展为集群



**分布式架构优点**

- 业务解耦，
- 系统模块化，可重用化
- 提升系统并发量
- 优化测试运维部署效率

**分布式架构缺点**

- 架构复杂
- 部署多个子系统复杂
- 系统之间通信耗时
- 新人融入团队缓慢
- 调试复杂

**分布式架构设计原则**

- 异步解耦 - 消息队列
- 幂等一致性
- 融合分布式中间件
- 容错高可用



## 1.2 为什么引入 Redis ?

![image-20211217013954553](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20211217013954553.png)

现有架构存在一个问题，当出现大流量请求时，所有的请求都会到达数据库。根据二八原则，**大多数的请求都是读请求**，我们可以引入 Redis 来作为数据库的屏障，把经常读取的数据缓存到 Redis，这样用户的很多请求就不会访问到数据库了，这样就可以提高数据库的读写效率，整个系统的吞吐量也随之提高。

![image-20211217015120420](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20211217015120420.png)



## 1.3 什么是 NoSql？

- Not Only Sql
- 传统项目使用关系数据库，单表性能有限
- 为互联网和大数据而生
- 横向扩展方便高效
- 键值对存储，增删非常高效
- 高性能读取，远超关系数据库
- 高可用
- 可以存数据，做缓存



NoSql 常见分类

- 键值对数据库  Redis Memcache
- 列存储数据库  Hbase  Cassandra
- 文档型数据库  MongoDB  CouchDB
- 图形数据库  Neo4J  FlockDB



分布式缓存

- 提升读取速度
- 分布式计算
- 为数据库降低查询压力
- 跨服务器缓存
- 内存缓存，相比磁盘存储提高并发量



## 1.4 技术选型

- EhCache
  - 基于 java 开发
  - 基于 JVM 缓存
  - 简单轻巧方便
  - 不支持集群
  - 不支持分布式
- MemCache
  - 键值对存储
  - 内存使用率比较高
  - 多核多线程
  - 无法容灾
  - 无法持久化
- Redis
  - 丰富的数据结构
  - 持久化
  - 主从同步、故障转移
  - 内存数据库
  - 单线程
  - 单核，无法利用多核 CPU



## 1.5 安装 Redis

`docker run -p 6379:6379 --name redis -v D:/local/docker/redis5.0.5/conf/redis.conf:/etc/redis/redis.conf -v D:/local/docker/redis5.0.5/data:/data -d redis:5.0.5 redis-server /etc/redis/redis.conf`





## 1.6 Redis 的数据类型



www.redisdoc.com   Redis 命令手册





# 2. SpringBoot 整合 Redis 实战

1. 引入 redis

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

2. 配置 redis 相关属性

```yml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    database: 1
    password: imooc
```

3. 编写 Redis 测试类，测试写入值，获取值

```java
@RestController
@RequestMapping(("redis"))
public class RedisController {
    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * http://localhost:8088/redis/set?key=aaa&value=123
     * 需要注意，通过这里写入redis的数据，key序列化会有问题
     */
    @GetMapping("/set")
    public String set(String key, String value) {
        redisTemplate.opsForValue().set(key, value);
        return "OK";
    }

    /**
     * http://localhost:8088/redis/get?key=aaa
     */
    @GetMapping("/get")
    public String get(String key) {
        String value = (String) redisTemplate.opsForValue().get(key);
        return value;
    }
```









# 3. Redis 进阶提升与主从复制



## 3.1 持久化

Redis 有两种持久化方式：RDB 和 AOF，在配置文件`redis.conf`中可以修改持久化方式，



**RDB** (Redis DataBase) 是定时备份，在指定的时间间隔下，将内存数据写入到磁盘作为快照，默认文件名为`/data/dump.rdb`。在恢复时将快照文件读入内存。

- 优点：
  1. 每隔一段时间备份，全量备份
  2. 灾备简单，可以远程传输
  3. 子进程备份的时候，主进程不会与任何 io 操作（写入修改和删除），保证数据的完整性

- 劣势：
  1. 发生故障时，有可能会丢失最后一次的备份数据
  2. 子进程所占用的呢欧村会和父进程一模一样，会造成 CPU 负担
  3. 定时全量备份是重量级操作，对于实时备份无法处理



RDB 会丢失最后一次备份后的数据，但由于是 Redis 保存的是缓存数据，影响不大。如果追求数据的完整性，则应该使用 AOF 持久化。

**AOF** (Append Of File) 以日志的形式来记录用户请求的写操作（不用记录读操作），文件以追加的形式进行记录，默认文件名为`/data/appendonly.aof`。在恢复时就是将文件从头到尾执行一次。

- 优点：
  1. AOF 可以以秒级别为单位进行备份，如果发生问题，也只会丢失最后一秒的数据，数据可靠性和完整性更好。AOF 每秒执行一次 fsync 操作进行备份
  2. 以 log 日志的形式追加记录，如果磁盘满了，会执行 redis-check-aof 工具
  3. 当数据太大的时候，redis 可以在后台自动重写 aof，不会影响客户端的读写操作
  4. AOF 日志包含所有写操作，便于 redis 的解析和恢复
- 劣势：
  1. 针对相同的数据进行备份，AOF 比 RDB 更大
  2. AOF 相比 RDB 慢



如何选择 RDB 和 AOF ？

1. 如果你能接受一段时间的缓存丢失，就使用 RDB
2. 如果你对实时性的数据比较关心，就使用 AOF
3. 可以使用 RDB 和 AOF 结合做持久化，RDB 做冷备，可以在不同时期对不同版本做恢复，AOF 做热备，保证数据仅有 1 秒的丢失。





## 3.2 主从架构



![image-20220113180615056](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220113180615056.png)





![image-20220113180743192](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220113180743192.png)

无磁盘化复制，不需要保存到磁盘后再传输，通过 socket 在内存中直接传输。



## 缓存过期机制

计算机内存优先，Redis 的数据是保存在内存中的，设置了expire的key缓存过期了，但是服务器的内存还是会被占用，这是因为redis所基于的两种删除策略

- 主动定期删除
  - 定时随机的检查key，如果 key 过期则清理删除。（每秒检查次数在redis.conf中的hz配置）
- 被动惰性删除
  - 当客户端请求一个已经过期的 key 的时候，那么 redis 会检查这个 key 是否过期，如果过期了，则删除，然后返回一个 nil。这种策略友好，不会有太多的 CPU  损耗，但是内存占用会比较高。



> 如果内存被 Redis 缓存占用满了怎么办 ?

虽然内存占满了可以使用硬盘来保存，但是没有意义，因为硬盘相比内存慢很多，影响 Redis 性能。所以，当内存满了后，Redis 提供了一套缓存淘汰机制：MEMORY MANAGEMENT

可以在`redis.conf` 中配置下列属性，来处理内存不够用的问题

```
maxmemory  当内存到达指定的使用率时,开始清理缓存
noeviction：旧缓存永不过期，新缓存设置不了，返回错误
allkeys-lru：清除最少用的旧缓存，然后保存新的缓存（推荐使用） 
allkeys-random：在所有的缓存中随机删除（不推荐） 
volatile-lru：在那些设置了expire过期时间的缓存中，清除最少用的旧缓存，然后保存新的缓存 
volatile-random：在那些设置了expire过期时间的缓存中，随机删除缓存 
volatile-ttl：在那些设置了expire过期时间的缓存中，删除即将过期的
```





# 4. Redis 哨兵机制与实现

> Redis 集群中，master 节点挂了，如果保证 Redis 可用，实现继续读写？

**什么是哨兵**

Sentinel(哨兵)是用于监控Redis集群中Master状态的工具，是 Redis 高可用解决方案，哨兵可以监视一个或者多个redis master服务，以及这些master服务的所有

某个master服务宕机后，会把这个master下的某个 slave 节点升级为master来替代已宕机的master继续工作。

![image-20220113183613222](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20220113183613222.png)

# 5. Redis 集群







# 6. 缓存雪崩与穿透



## 6.1 缓存穿透

**缓存穿透**：当每次请求数据，总是在redis中无法查到对应数据，都需要去查询 MySQL，即出现了缓存穿透。

举个例子，商品分类保存在 redis 中，当根据一个错误的商品分类 id，查询一个不存在的商品分类时返回 null，程序员因为结果是 null 则没有将这对数据保存到 redis。那么恶意请求，就可以一直查询该 id 的商品类目，提升 MySQL 的负载，导致 redis 屏障的作用荡然无存，这就叫**缓存穿透**

![image-20211217015120420](https://gitee.com/tracccer/picture-bed/raw/master/img/image-20211217015120420.png)

```java
@GetMapping("/subCat/{rootCatId}")
public IMOOCJSONResult subCat(
    @PathVariable(value = "rootCatId", required = true) Integer rootCatId) {
    // 这里不需要做 rootCatId 判空校验, 注解会自动校验
    List<CategoryVO> subCategories = new ArrayList<>();
    String subCategoryStr = redisOperator.get("subCategory:" + rootCatId);

    if (StringUtils.isEmpty(subCategoryStr)) {
        // 从数据库查找分类，并加入缓存
        subCategories = categoryService.getSubCatList(rootCatId);

        // 注意, 即使这里从数据库查到的subCategories为空, 也应该将其保存到redis中, 防止缓存穿透
        if (subCategories.isEmpty()) {
            redisOperator.set("subCategory:" + rootCatId, JsonUtils.objectToJson(subCategories), 5 * 60);
        } else {
            redisOperator.set("subCategory:" + rootCatId, JsonUtils.objectToJson(subCategories));
        }
    } else {
        subCategories = JsonUtils.jsonToList(subCategoryStr, CategoryVO.class);
    }

    return IMOOCJSONResult.ok(subCategories);
}
```







## 6.2 布隆过滤器









## 6.3 缓存雪崩



**缓存雪崩**：指缓存由于某些原因（比如 宕机、cache服务挂了或者不响应）整体crash掉了，导致大量请求到达后端数据库，从而导致数据库崩溃，整个系统崩溃，发生灾难。



雪崩预防的几种方式：

- 永不过期
- 过期时间错开
- 多缓存结合
- 使用第三方的 Redis 云服务



## 6.4 批量查询优化



- multiGet  批量查询

  ```java
  public List<String> mget(List<String> keys) {
      return redisTemplate.opsForValue().multiGet(keys);
  }
  ```

  

- pipeline  通过管道实现批量查询/写入

  ```java
  public List<Object> batchGet(List<String> keys) {
      List<Object> result = redisTemplate.executePipelined(new RedisCallback<String>() {
          @Override
          public String doInRedis(RedisConnection connection) throws DataAccessException {
              StringRedisConnection src = (StringRedisConnection)connection;
  
              for (String k : keys) {
                  src.get(k);
              }
              return null;
          }
      });
  
      return result;
  }
  ```

  



# 面试题

参考 1-8 章节





# 参考文档

1. [Redis6 入门到精通 - 尚硅谷](https://www.bilibili.com/video/BV1Rv41177Af) 10小时，详细学习 Redis 看这个吧



# 学习记录

2022年1月13日15:38:37
