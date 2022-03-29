

# 1 zookeeper 与分布式系统

zookeeper 是一个中间件，为**分布式系统**提供**协调（Coordination）服务**。是Google Chubby的开源实现，Google的三篇论文总都提及了一个lock service -- Chubby，于是就有了Chubby的开源实现 zookeeper。

## 1.1 什么是分布式系统

**分布式系统**

- 很多台计算机组成一个整体，一个整体一致对外并且处理同一个请求。

- 内部的每台计算机都可以相互通信（rest/RPC）

- 客户端到服务端的一次请求，到响应结束会历经多台计算机

如下图所示，小慕是客户端，访问分布式文件系统（网盘），服务端在服务器A，服务器B是服务器A的备用机从而实现高可用，具体的文件保存在文件服务器中，为了防止文件丢失，一个文件会保存在多个文件服务器中。一次请求历经了3台计算机。

![分布式系统图解1](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/分布式系统图解1.png)

下图是一个电商网站下单请求响应流程。用户在商品页面下单后，会经过商品服务查看商品库存，再经过订单服务生成订单，再经过账单服务，最后返回到商品页面。一个请求历经了4台计算机。

![分布式系统图解2](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/分布式系统图解2.png)

## 1.2 什么是zookeeper

zookeeper 是一个中间件，为**分布式系统**提供**协调服务**。我们可以把zookeeper看成是一个分布式数据库：

- 一个具有文件系统特点的分布式数据库
- 解决了数据一致性问题的分布式数据库
- 具有发布订阅功能的分布式数据库

##1.3 zookeeper的特性


- 一致性：数据一致性，数据按照顺序分批入库
- 原子性：事务要么成功要么失败
- 单一视图：客户端连接集群中的zk节点，数据都是一致的
- 可靠性：每次对zk的操作状态都会保存在服务端
- 实时性：客户端可以读取到zk服务端的最新数据



# 2 zookeeper 的安装与集群配置

1. 安装`JDK`，配置`JAVA_HOME`
2. 在[官网](zookeeper.apache.org)下载zookeeper压缩包，上传到Linux机器 /opt 目录
3. 解压，`tar -zxvf zookeeper3.4.10.tar.gz ` `cp -r /opt/zookeeper3.4.10.tar.gz /myzookeeper`
4. 修改zoo.cfg文件
5. 启动zookeeper服务端，`zkServer.sh start`
6. 检查是否启动成功，`ps -ef | grep zookeeper`检查进程， `echo ruok | nc 127.0.0.1:2181`返回imok

查看[官方文档](http://zookeeper.apache.org/doc/r3.1.2/zookeeperStarted.html)或zookeeper docs目录的index.html，Started Guide快速使用中介绍了单机安装和一些zk的基本概念，Programmer's Guide详细介绍了zk的数据模型，节点类型，会话，Watch事件，ACL权限控制等。



zookeeper 目录结构
----

bin：主要的一些运行命令，zkCli.sh 是启动zk客户端，zkServer.sh是启动zk服务端

conf：配置文件，我们需要修改zoo_sample.cfg

contrib：附加的一些功能

dist-maven：保存mvn编译结果的目录，包括jar，sources.jar，pom.xml

docs：zk帮助文档，可以打开index.html查看，与官网文档相同

lib：开发时使用的jar包，

recipes：案例demo代码，包括election，lock，queue

src：zk源码



zoo.cfg 配置
----
复制 conf 目录下的zoo_sample.cfg，重命名为zoo.cfg。该配置文件中有以下几个属性：

- tickTime：用于计算的时间单元，单位是毫秒。比如session超时设置为 N，则超时时间为N * tickTime

- initLimit：用于集群，初始化连接时间。follower服务器启动过程中，需要连接并同步Leader节点的所有最新数据，不能超过initLimit，以tickTime的倍数来表示

- syncLimit：用于集群，限制了**follower服务器与Leader服务器**之间请求和应答的时限（**心跳机制**）；如果A发出心跳包在syncLimit之后没有收到B的响应，就认为这个B已经不在线了
- dataDir：zookeeper存储的数据文件目录。dataDir=/usr/local/zookeeper/dataDir
- dataLogDir：**日志目录**，如果不配置则使用dataDir。dataLogDir==/usr/local/zookeeper/dataLogDir
- clientPort：客户端连接服务器的端口，**默认2181**

zk 的常用命令
----
`zkServer.sh start`  启动zk服务，在windows中是`zkServer.cmd`，不需要start命令。

`zkServer.sh stop`  停止zk服务

`zkServer.sh status`  查看zookeeper状态，返回zk的配置文件，客户端连接端口，服务器类型Mode为Leader或Follower

```bash
zkServer.sh status                                      
ZooKeeper JMX enabled by default
Using config: /opt/apache-zookeeper-3.5.5-bin/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost.
Mode: leader
```

`echo ruok | nc 127.0.0.1:2181` 返回imok说明zkServer启动成功

`jps` 查看启动的java进程，zk进程名称为QuorumPeerMain

`zkCli.sh`  启动zk客户端，集群状态需要制定zk服务器`zkCli.sh -server 192.168.100.1:2181`

zookeeper服务启动日志如下所示，主要包括以下5部分内容：

1. 以单机模式启动`running  in standalone mode`，
2. 读取配置文件zoo.cfg`Reading configuration from: E:\zookeeper-3.4.10\bin\..\conf\zoo.cfg`，
3. 开始启动服务`Starting server`，
4. 显示zk环境信息，包括zk的版本号 version，主机名称 hostname，java 版本，java_home，classpath，操作系统，用户名称等
5. 显示配置信息，包括`tickTime set to 2000`，绑定端口`binding to port 0.0.0.0/0.0.0.0:2181`



```
E:\zookeeper-3.4.10\bin>zkServer.cmd

E:\zookeeper-3.4.10\bin>call "C:\Program Files\Java\jdk1.8.0_51"\bin\java "-Dzookeeper.log.dir=E:\zookeeper-3.4.10\bin\.." "-Dzookeeper.root.logger=INFO,CONSOLE" -cp "E:\zookeeper-3.4.10\bin\..\build\classes;E:\zookeeper-3.4.10\bin\..\build\lib\*;E:\zookeeper-3.4.10\bin\..\*;E:\zookeeper-3.4.10\bin\..\lib\*;E:\zookeeper-3.4.10\bin\..\conf" org.apache.zookeeper.server.quorum.QuorumPeerMain "E:\zookeeper-3.4.10\bin\..\conf\zoo.cfg"
2020-02-04 11:31:42,927 [myid:] - INFO  [main:QuorumPeerConfig@134] - Reading configuration from: E:\zookeeper-3.4.10\bin\..\conf\zoo.cfg
2020-02-04 11:31:42,938 [myid:] - INFO  [main:DatadirCleanupManager@78] - autopurge.snapRetainCount set to 3
2020-02-04 11:31:42,939 [myid:] - INFO  [main:DatadirCleanupManager@79] - autopurge.purgeInterval set to 0
2020-02-04 11:31:42,940 [myid:] - INFO  [main:DatadirCleanupManager@101] - Purge task is not scheduled.
2020-02-04 11:31:42,943 [myid:] - WARN  [main:QuorumPeerMain@113] - Either no config or no quorum defined in config, running  in standalone mode
2020-02-04 11:31:43,037 [myid:] - INFO  [main:QuorumPeerConfig@134] - Reading configuration from: E:\zookeeper-3.4.10\bin\..\conf\zoo.cfg
2020-02-04 11:31:43,039 [myid:] - INFO  [main:ZooKeeperServerMain@96] - Starting server
2020-02-04 11:31:52,125 [myid:] - INFO  [main:Environment@100] - Server environment:zookeeper.version=3.4.10-39d3a4f269333c922ed3db283be479f9deacaa0f, built on 03/23/2017 10:13 GMT
2020-02-04 11:31:52,125 [myid:] - INFO  [main:Environment@100] - Server environment:host.name=DESKTOP-HSRU97J
2020-02-04 11:31:52,128 [myid:] - INFO  [main:Environment@100] - Server environment:java.version=1.8.0_51
2020-02-04 11:31:52,129 [myid:] - INFO  [main:Environment@100] - Server environment:java.vendor=Oracle Corporation
2020-02-04 11:31:52,130 [myid:] - INFO  [main:Environment@100] - Server environment:java.home=C:\Program Files\Java\jdk1.8.0_51\jre
2020-02-04 11:31:52,130 [myid:] - INFO  [main:Environment@100] - Server environment:java.class.path=E:\zookeeper-3.4.10\bin\..\build\classes;E:\zookeeper-3.4.10\bin\..\build\lib\*;E:\zookeeper-3.4.10\bin\..\zookeeper-3.4.10.jar;E:\zookeeper-3.4.10\bin\..\lib\jline-0.9.94.jar;E:\zookeeper-3.4.10\bin\..\lib\log4j-1.2.16.jar;E:\zookeeper-3.4.10\bin\..\lib\netty-3.10.5.Final.jar;E:\zookeeper-3.4.10\bin\..\lib\slf4j-api-1.6.1.jar;E:\zookeeper-3.4.10\bin\..\lib\slf4j-log4j12-1.6.1.jar;E:\zookeeper-3.4.10\bin\..\conf
2020-02-04 11:31:52,131 [myid:] - INFO  [main:Environment@100] - Server environment:java.library.path=C:\Program Files\Java\jdk1.8.0_51\bin;C:\WINDOWS\Sun\Java\bin;C:\WINDOWS\system32;C:\WINDOWS;C:\WINDOWS\system32;C:\WINDOWS;C:\WINDOWS\System32\Wbem;C:\WINDOWS\System32\WindowsPowerShell\v1.0\;C:\Program Files\Java\jdk1.8.0_51\bin;C:\Program Files\Java\jdk1.8.0_51\jre\bin;E:\Program Files (x86)\apache-maven-3.3.9\bin;C:\WINDOWS\System32\OpenSSH\;E:\Program Files (x86)\Git\cmd;C:\Users\Administrator\AppData\Local\Microsoft\WindowsApps;C:\Users\Administrator\AppData\Local\GitHubDesktop\bin;%USERPROFILE%\AppData\Local\Microsoft\WindowsApps;;.
2020-02-04 11:31:52,132 [myid:] - INFO  [main:Environment@100] - Server environment:java.io.tmpdir=C:\Users\ADMINI~1\AppData\Local\Temp\
2020-02-04 11:31:52,133 [myid:] - INFO  [main:Environment@100] - Server environment:java.compiler=<NA>
2020-02-04 11:31:52,138 [myid:] - INFO  [main:Environment@100] - Server environment:os.name=Windows 8.1
2020-02-04 11:31:52,139 [myid:] - INFO  [main:Environment@100] - Server environment:os.arch=amd64
2020-02-04 11:31:52,141 [myid:] - INFO  [main:Environment@100] - Server environment:os.version=6.3
2020-02-04 11:31:52,144 [myid:] - INFO  [main:Environment@100] - Server environment:user.name=Administrator
2020-02-04 11:31:52,145 [myid:] - INFO  [main:Environment@100] - Server environment:user.home=C:\Users\Administrator
2020-02-04 11:31:52,146 [myid:] - INFO  [main:Environment@100] - Server environment:user.dir=E:\zookeeper-3.4.10\bin
2020-02-04 11:31:52,158 [myid:] - INFO  [main:ZooKeeperServer@829] - tickTime set to 2000
2020-02-04 11:31:52,158 [myid:] - INFO  [main:ZooKeeperServer@838] - minSessionTimeout set to -1
2020-02-04 11:31:52,160 [myid:] - INFO  [main:ZooKeeperServer@847] - maxSessionTimeout set to -1
2020-02-04 11:31:52,301 [myid:] - INFO  [main:NIOServerCnxnFactory@89] - binding to port 0.0.0.0/0.0.0.0:2181
2020-02-04 11:32:01,829 [myid:] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:NIOServerCnxnFactory@192] - Accepted socket connection from /0:0:0:0:0:0:0:1:60100
2020-02-04 11:32:01,852 [myid:] - INFO  [NIOServerCxn.Factory:0.0.0.0/0.0.0.0:2181:ZooKeeperServer@942] - Client attempting to establish new session at /0:0:0:0:0:0:0:1:60100
2020-02-04 11:32:01,862 [myid:] - INFO  [SyncThread:0:FileTxnLog@203] - Creating new log file: log.7a8
2020-02-04 11:32:02,218 [myid:] - INFO  [SyncThread:0:ZooKeeperServer@687] - Established session 0x1700e411a110000 with negotiated timeout 30000 for client /0:0:0:0:0:0:0:1:60100
```







# 3 zookeeper基本数据模型

zookeeper数据模型是一个树形结构，类似于linux文件结构。如下图所示，zk根目录是 / ，是一个树形结构

![zk结构](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/zk结构.png)


![zk结构2](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/zk结构2.png)

- zk的数据模型可以理解为linux的文件目录：/usr/local/...

- 每一个节点都称为**znode**，znode可以有子节点，也可以有数据。

- 每个节点分为**临时节点**和**永久节点**，临时节点在客户端断开后消失  

- 每个节点znode都有自己的**版本号**，可以通过命令行来显示节点信息

- 每当节点数据发生变化，那么该节点的**版本号会加1**（乐观锁^参考文档1^）

- 删除/修改过时节点时，因为版本号不匹配，则会修改失败（乐观锁^参考文档1^）

- 每个节点znode存储的数据不宜过大，几k即可

- 节点可以设置权限控制列表acl，可以通过权限设置来限制用户的访问



## 3.1 zk 数据模型基本操作


**客户端连接**

使用命令`zkCli.sh`启动客户端，启动成功信息如下，表示连接到了 localhost:2181，连接状态是CONNECTED，后面的数字 0 表示运行的命令数

```bash
[zk: localhost:2181(CONNECTED) 0]
```

输入`help`命令，查看zk客户端的常用命令如下，  

```
ZooKeeper -server host:port cmd args
        stat path [watch]
        set path data [version]
        ls path [watch]
        delquota [-n|-b] path
        ls2 path [watch]
        setAcl path acl
        setquota -n|-b val path
        history
        redo cmdno
        printwatches on|off
        delete path [version]
        sync path
        listquota path
        rmr path
        get path [watch]
        create [-s] [-e] path data acl
        addauth scheme auth
        quit
        getAcl path
        close
        connect host:port
```



## znode结构

Znode由三部分组成：path，data，Stat

```bash
[zk: ] get /zookeeper	# 节点路径path
				# 节点保存的数据data,此节点数据为空
# 下面是Stat信息
cZxid = 0x0 	# 节点创建操作的zxid，create
ctime = Thu Jan 01 08:00:00 CST 1970	# 创建节点时间
mZxid = 0x0 	# 节点最新修改操作的zxid，modify 
mtime = Thu Jan 01 08:00:00 CST 1970	# 修改节点时间
pZxid = 0x0 	# 子节点最后更新的zxid
cversion = -1	# 子节点的修改次数，每次修改子节点version会加1, children  
dataVersion = 0	# 当前节点保存的数据的修改次数，每次修改数据version会加1
aclVersion = 0	# 用户控制权限的修改次数，每次修改权限version会加1
ephemeralOwner = 0x0	# 如果是临时节点表示该节点的session id；非临时节点则为0
dataLength = 0	# 节点保存的数据的大小
numChildren = 1	# 子节点的数量
```





## 3.2 zookeeper的应用场景


1. **统一配置文件管理**，即只需要部署一台服务器，则可以把相同的配置文件同步更新到其他所有服务器，此操作在云计算中应用特别多。<a href="#watchconfig">查看详细 </a>
2. **服务注册和发现**，类似消息队列MQ，dubbo发布者会把数据存到znode中，订阅者会读取这个数据。如下图所示，发布者发布数据，订阅者根据数据的变化进行操作。利用 Znode 和 Watcher，可以实现分布式服务的注册和发现，最著名的应用就是阿里的分布式 RPC 框架 Dubbo。
![发布订阅.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/发布订阅.png)
1. **提供分布式锁**，分布式环境中不同进程之间会争夺资源，类似多线程中的锁。下图中多个服务器中的进程要操作网盘中的文件，为了避免冲突，需要分布式锁。雅虎研究员设计 ZooKeeper 的初衷。利用 ZooKeeper 的临时顺序节点，可以轻松实现分布式锁。![分布式锁.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/分布式锁.png)
2. 集群管理，集群中保证数据的强一致性。无论客户端读取哪一台机器的数据，都会得到一致的数据，因为zookeeper会将数据从主节点同步到其他节点
3. 此外，Kafka、HBase、Hadoop 也都依靠 ZooKeeper 同步节点信息，实现高可用。



# 4 zookeeper 基本特性与客户端操作

## 4.1 session的基本原理

1. 客户端与服务端之间的的连接称之为Session(会话)
2. 每个Session会话都可以设置一个超时时间，超时后Session会被销毁
3. 心跳停止，则session过期
4. Session过期，则临时节点znode会被抛弃
5. 心跳机制，客户端向服务端的ping包请求，为了向服务端表示客户端在线



## 4.2 常用命令行操作

zookeeper节点znode有许多状态信息(Stat)，其中有两个重要概念`zxid`和`version numbers`。

**zxid**

下面的命令中经常出现`Zxid`，对ZooKeeper节点和子节点创建、更新数据（查询不会修改zxid）都会收到一个zxid **(ZooKeeper Transaction Id)**形式的标记。这将向ZooKeeper公开所有更改的总顺序。每次更改都有一个惟一的zxid，如果zxid1小于zxid2，则说明zxid1发生在zxid2之前。

zookeeper中的操作分为事务性操作（`create，set，delete`），会使得`zxid`加1，并且将该操作记录**持久化**到日志中；而非事务性操作（`get，exist`）不会修改`zxid`。

**Version Numbers**

对节点的每次更改都会使得该节点的版本号version加 1。总共有三个version：

1. `version`：对znode的数据的更改次数
2. `cversion`：对znode的子节点的更改次数
3. `aclversion`：对znode的ACL的更改次数

**常用命令**

`ls`：显示指定目录（节点）下的子节点

`ls2`：ls2是显示指定目录（节点）下的子节点，指定目录的**状态信息**。等同于`ls`+`stat`命令

```shell
[zk: localhost:2181(CONNECTED) 2] ls /zookeeper
[quota]

[zk: localhost:2181(CONNECTED) 4] ls2 /zookeeper
[quota]
cZxid = 0x0 	# 节点创建操作的zxid，create
ctime = Thu Jan 01 08:00:00 CST 1970	# 创建节点时间
mZxid = 0x0 	# 节点最新修改操作的zxid，modify 
mtime = Thu Jan 01 08:00:00 CST 1970	# 修改节点时间
pZxid = 0x0 	# 子节点最后更新的zxid
cversion = -1	# 子节点的修改次数，每次修改子节点version会加1, children  
dataVersion = 0	# 当前节点保存的数据的修改次数，每次修改数据version会加1
aclVersion = 0	# 用户控制权限的修改次数，每次修改权限version会加1
ephemeralOwner = 0x0	# 临时节点拥有者,如果是临时节点表示该节点的会话id；非临时节点则为0
dataLength = 0	# 节点保存的数据的大小
numChildren = 1	# 子节点的数量
```

`stat`：显示指定节点的状态信息，与`get`命令的区别是**不显示保存的数据信息**

`create [-s] [-e] path data acl`：创建节点，-e Ephemeral表示临时节点，-s sequence表示顺序节点，**data必填，否则无法创建**，不支持递归创建

`get`：获取指定**节点保存的信息**和状态信息

```bash 
[zk:] create -e /imooc/tmp imooc-data2
Created /imooc/tmp

[zk: localhost:2181(CONNECTED) 9] get /imooc
imooc-data		# 节点保存的数据信息
cZxid = 0x7ab
ctime = Tue Feb 04 16:11:01 CST 2020
pZxid = 0x7ac	# 子节点最新操作的zxid
cversion = 1	# 子节点的修改次数
ephemeralOwner = 0x0	# 非临时节点,所以为0
numChildren = 1			# 创建了1个子节点,所以为1

[zk: localhost:2181(CONNECTED) 10] get /imooc/tmp
imooc-data2	
cZxid = 0x7ac	# 节点创建操作的zxid,与父节点的pZxid相同
ephemeralOwner = 0x1700e411a110001		# 临时节点,表示会话id
```

> **问题**：创建临时节点后，停止客户端，该临时节点会立即消失吗？

使用客户端A创建临时节点ephNode，客户端B可以查看该临时节点，强行终止客户端A（不能使用quit命令退出），发现客户端B**仍然能够查看该临时节点**，因为心跳存在超时时间，在超时范围内，zk认为该客户端仍然正常。

当心跳超时后，session会话过期，临时节点ephNode 也会被抛弃，此时使用客户端B就查看不到该临时节点了。查看`zoo.cfg`文件，`syncLimit`属性就是心跳超时时间

```bash
# create -s 表示创建序列自增节点,设置的节点名称后会添加自增数
[zk:] create -s /imooc/seq seq-data
Created /imooc/seq0000000005
[zk:] create -s /imooc/seque seq-data
Created /imooc/seque0000000006
```

`set path data [version]`：设置节点的数据 ，version表示修改指定`dataversion`的数据，如果参数version与节点的`dataversion`不一致，则修改失败，这是为了避免多个客户端同时修改数据竞争产生的问题。

```bash
[zk: localhost:2181(CONNECTED) 3] get /imooc
imooc-data
mZxid = 0x7ab
dataVersion = 0

[zk: localhost:2181(CONNECTED) 4] set /imooc new-data
mZxid = 0x7c0		# 修改了节点数据,记录修改操作的zxid
dataVersion = 1		# 修改节点数据的次数

#修改节点指定版本的数据
[zk: localhost:2181(CONNECTED) 6] set /imooc 123 1
dataVersion = 2

# 当节点dataVersion与参数1不相等时,则修改失败.乐观锁
[zk: localhost:2181(CONNECTED) 7] set /imooc 123 1
version No is not valid : /imooc
```

`delete path [version]`：删除节点，version需要与节点`dataversion`一致，否则删除失败

## 4.3 watcher机制

客户端可以在节点znode上设置一个watch事件，对该znode的更改将触发该watch事件，并清除该watch事件。当一个watch事件触发时，zookeeper会向客户端发送一个通知。watcher机制的特点如下所示：

- 针对每个节点znode的操作，都会有一个监督者`watcher`

- 当监控的某个节点znode发生了变化，则**触发watcher事件**（类似触发器）

- watcher是**一次性**的，触发后以及销毁

- 节点znode**自己**、**子孙节点的创建、删除、数据修改**都能触发当前节点的watcher。节点没有创建之前也能添加watcher

- 不同类型的操作，触发不同的watcher事件，包括节点**创建、删除、数据修改**事件

  - 创建自身节点触发：NodeCreated

  - 修改自身节点数据触发：NodeDataChanged

  - 删除自身节点触发：NodeDeleted

  - 创建、删除子节点都会触发：NodeChildrenChanged

  - 修改子节点数据不会触发watch事件


`stat path [watch]:`获取节点状态信息，给节点**添加一次性的watch事件**

`get path [watch]:`获取节点状态信息和数据信息，给节点**添加一次性的watch事件**

`ls2 path [watch]:`获取节点状态信息和子节点，给节点**添加一次性的watch事件**

下面代码演示给**节点mywatch**添加watch事件，创建、删除、数据修改**节点mywatch自己**会触发哪些类型的watch事件：

```bash
# 给不存在的节点mywatch添加watch事件
[zk: localhost:2181(CONNECTED) 14] stat /mywatch watch
Node does not exist: /mywatch
# 创建节点mywatch，触发watch事件WatchedEvent，类型是NodeCreated
[zk: localhost:2181(CONNECTED) 15] create /mywatch 123

WATCHER::
Created /mywatch
WatchedEvent state:SyncConnected type:NodeCreated path:/mywatch

# 因为watch事件是一次性的，所以我们重新添加watch事件
[zk: localhost:2181(CONNECTED) 19] get /mywatch watch
123
# 修改节点数据，触发WatchedEvent，类型是NodeDataChanged
[zk: localhost:2181(CONNECTED) 20] set /mywatch 456

WATCHER::ctime = Thu Feb 06 12:22:27 CST 2020
WatchedEvent state:SyncConnected type:NodeDataChanged path:/mywatchmZxid = 0x7c9

# 再次添加watch事件
[zk: localhost:2181(CONNECTED) 21] get /mywatch watch
456

# 删除节点mywatch,触发WatchedEvent,类型是NodeDeleted
[zk: localhost:2181(CONNECTED) 22] delete /mywatch

WATCHER::
[zk: localhost:2181(CONNECTED) 23]
WatchedEvent state:SyncConnected type:NodeDeleted path:/mywatch
```

创建、删除子节点会触发NodeChildrenChanged事件，但**修改子节点数据不会触发watch事件**

```bash
# 创建节点mywatch
[zk: localhost:2181(CONNECTED) 31] create /mywatch 123
Created /mywatch
# 给节点添加watch事件
[zk: localhost:2181(CONNECTED) 33] ls /mywatch watch
[]
# 创建mywatch的子节点,触发WatchedEvent,类型是NodeChildrenChanged
[zk: localhost:2181(CONNECTED) 34] create /mywatch/cnode 666

WATCHER::Created /mywatch/cnode    # 这里是创建的节点路径
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/mywatch  	# 这里是触发watch事件的节点自身
```

watcher是当前客户端加在节点znode上的触发器，



watcher使用场景
----
- 统一配置文件管理。sqlConfig节点保存json数据，即对配置文件的操作和文件路径，某个客户端对sqlConfig节点添加了watch事件，当节点数据更新后，所有客户端都能监听到，然后根据节点数据更新本地配置信息。<a href="#watchconfig">第7.2章节 </a>详细介绍了利用Watch实现统一配置文件管理。

![统一配置信息.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/统一配置信息.png)

## 4.4 ACL权限控制

ACL(access control lists)

- 针对节点可以设置相关读写权限，保障数据安全性
- 权限permissions可以指定不同的权限范围和角色
- zk的acl通过[schema: id : permissions] 来构成权限列表；
  - schema：权限机制，有五种类型：world，auth，digest，ip，super
  - permissions：创建、删除、读、写权限
  - id：用户，permissions：权限组合字符串

身份认证的5种类型schema
----

world：默认方式，相当于全世界都能访问，只有一个id anyone	`world:anyone:[permissions]`；

auth：代表节点授权的用户  `auth:username:password:cdrwa`

digest：即用户名:密码这种方式认证，这也是业务系统中最常用的，`digest:username:BSE64(SHA1(password)):[permissions]`

ip：指定的ip地址才可以访问，`ip:182.168.1.1:[permissions]`

super：超级管理员，拥有所有权限，需要修改`zkServer.sh`文件

permissions
---

权限字符串缩写`crdwa`

- create：创建子节点
- read：获取节点 / 子节点信息和数据
- delete：删除子节点
- write：设置节点数据
- admin：管理权限，设置节点ACL的权限

访问
-----

`addauth digest user:pwd` 来添加当前上下文中的授权用户，`auth`和`digest`两种授权方式均可以通过`addauth digest user:pwd`命令（明文密码）访问。



登录后设置权限可省略username和password

```bash
# 未添加授权addauth, 设置ACL失败
[zk: localhost:2181(CONNECTED) 7] setAcl /myacl auth:mao:mao:crdwa
Acl is not valid : /myacl
# 添加授权用户mao:mao
[zk: localhost:2181(CONNECTED) 8] addauth digest mao:mao
[zk: localhost:2181(CONNECTED) 9] setAcl /myacl auth:mao:mao:crwa
aclVersion = 1
[zk: localhost:2181(CONNECTED) 12] setAcl /myacl auth:mao:123456:crdwa
aclVersion = 2

# ACL列表仍然只有 mao:mao, 没有mao:123456
[zk: localhost:2181(CONNECTED) 13] getAcl /myacl
'digest,'mao:LVVsVUii7a7fmrx8wQgjm3ljkTA=
: crwa
# 省略username和password, 使用当前授权的用户, 修改权限为crdwa
[zk: localhost:2181(CONNECTED) 14] setAcl /myacl auth:::crdwa
aclVersion = 3
# 修改成功
[zk: localhost:2181(CONNECTED) 29] getAcl /myacl
'digest,'mao:LVVsVUii7a7fmrx8wQgjm3ljkTA=
: cdrwa
```



ACL命令行
----

- `getAcl:`获取某个节点的acl权限信息，`getAcl /imooc/myauth`
- `setAcl:`设置某个节点的acl权限信息，`setAcl /imooc/myauth auth:mao:mao:cdrwa`
- `addauth:` 来添加当前上下文中的授权用户，`addauth digest mao:maos`

```bash
# 修改myauth节点的acl权限为crwa,即无法删除子节点
[zk: localhost:2181(CONNECTED) 26] setAcl /imooc/myauth world:anyone:crwa
aclVersion = 1
# 创建myauth的子节点test
[zk: localhost:2181(CONNECTED) 27] create /imooc/myauth/test 222
Created /imooc/myauth/test
# 删除myauth子节点test,发生权限错误
[zk: localhost:2181(CONNECTED) 28] delete /imooc/myauth/test
Authentication is not valid : /imooc/myauth/test

# 添加当前上下文中的授权用户，相当于登录，否则下面的setAcl命令会失败
[zk: localhost:2181(CONNECTED) 31] addauth digest mao:mao
# 使用用户mao设置acl权限, 当用户名密码不是当前用户mao:mao时不生效
# 和下行命令等价 setAcl /imooc/myauth auth:::cdrwa
[zk: localhost:2181(CONNECTED) 32] setAcl /imooc/myauth auth:mao:mao:cdrwa
aclVersion = 2
# 查看myauth权限
[zk: localhost:2181(CONNECTED) 33] getAcl /imooc/myauth
'digest,'mao:LVVsVUii7a7fmrx8wQgjm3ljkTA=
: cdrwa
[zk: localhost:2181(CONNECTED) 34]

# 使用digest设置acl权限
[zk: localhost:2181(CONNECTED) 34] setAcl /imooc/myauth digest:mao:LVVsVUii7a7fmrx8wQgjm3ljkTA=:cdra
aclVersion = 3
# 查看myauth权限,发现已修改
[zk: localhost:2181(CONNECTED) 35] getAcl /imooc/myauth
'digest,'mao:LVVsVUii7a7fmrx8wQgjm3ljkTA=
: cdra
# 修改节点数据,提示权限不合法
[zk: localhost:2181(CONNECTED) 36] set /imooc/myauth 222
Authentication is not valid : /imooc/myauth
```

**super 权限设置**

修改zkServer.sh文件，添加系统属性“-Dzookeeper.DigestAuthenticationProvider.superDigest=username:BASE64(SHA1(password))”。zk会读取该属性并设置为super用户，源码如下图所示

![super用户.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/super用户.png)


## 4.5 四字命令

四字命令是在**Linux中使用**（zkCli无法使用）来zookeeper服务的当前状态及相关信息的，四字命令的

zk可以通过它自身提供的简写命令来和服务器交互，需要使用到`nc`命令，需要使用`yum install nc`安装，命令格式为 `echo [commond] | nc [ip] [port]`

- stat：查看zk的状态信息和Mode类型
- ruok：查看当前zkServer是否启动，若启动成功则返回 imok
- dump：列出未经处理的会话和临时节点
- conf：查看服务配置
- cons：展示连接到服务器的客户端信息
- envi：环境变量
- mntr：监控zk健康信息
- wchs：展示watch的详细信息
- wchc：通过session列出服务器watch的详细信息，
- wchp：通过路径列出服务器 watch的详细信息

# 5 zookeeper集群安装







# 6 zookeeper JavaAPI开发客户端

1. 依赖

   使用zookeeper 原生JavaAPI开发需要引入相应的jar包，依赖`pom.xml`如下所示：

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.7</version>
</dependency>
```

## 6.1 会话连接与恢复（源码）

建立客户端与zk服务端的session连接，需要以下三步：

1. 需要创建zk对象，**传入Watcher对象**
2. 启动`sendThread`线程与zk服务端建立连接
3. 启动`eventThread`线程，不断检查连接是否建立成功；若成功，则触发watch事件，使用Watcher发送watch通知。

```java
// ZKConnect是Watcher接口的实现类，用于发送watch通知，即调用Watch.process()方法
ZooKeeper zk = new ZooKeeper(zkServerPath, timeout, new ZKConnect());
```

客户端连接zk服务端代码如下所示：

 ```java
// 实现Watcher接口,用于通知客户端是否连接成功
public class ZKConnect implements Watcher {

    final static Logger log = LoggerFactory.getLogger(ZKConnect.class);

    public static final String zkServerPath = "localhost:2181";
    //	public static final String zkServerPath = "192.168.1.111:2181,192.168.1.111:2182,192.168.1.111:2183";
    public static final Integer timeout = 5000;

    public static void main(String[] args) throws Exception {
        /**
         * 客户端和zk服务端链接是一个异步的过程
         * 当连接成功后后，客户端会收的一个watch通知，即调用Watch.process()方法
         *
         * 参数：
         * connectString：连接服务器的ip字符串，多个ip用逗号分隔
         * sessionTimeout：超时时间，心跳收不到了，那就超时
         * watcher：通知事件，如果有对应的事件触发，则会收到一个通知；如果不需要，那就设置为null
         * sessionId：会话的id
         * sessionPasswd：会话密码	当会话丢失后，可以依据 sessionId 和 sessionPasswd 重新获取会话
         */
        ZooKeeper zk = new ZooKeeper(zkServerPath, timeout, new ZKConnect());

        log.warn("客户端开始连接zookeeper服务器...");
        log.warn("连接状态：{}", zk.getState());

        // 等待连接线程执行完毕
        new Thread().sleep(2000);

        log.warn("连接状态：{}", zk.getState());
    }

    // 连接成功后使用watch事件进行通知
    @Override
    public void process(WatchedEvent event) {
        log.warn("接受到watch通知：{}", event);
    }
}
 ```

会话恢复

将之前创建的zk连接会话的`sessionId`和`sessionPasswd`保存，然后利用其创建新的

zk对象即可恢复会话，[查看完整代码](github.com)

```java
	ZooKeeper zk = new ZooKeeper(zkServerPath, timeout, new ZKConnectSessionWatcher());
		
		long sessionId = zk.getSessionId();
		byte[] sessionPassword = zk.getSessionPasswd();
		
		log.warn("客户端开始连接zookeeper服务器...");
		log.warn("连接状态：{}", zk.getState());
		new Thread().sleep(1000);
		log.warn("连接状态：{}", zk.getState());
		
		new Thread().sleep(200);
		
		// 开始会话重连,使用之前保存的sessionId和password创建新的连接
		ZooKeeper zkSession = new ZooKeeper(zkServerPath, 
											timeout, 
											new ZKConnectSessionWatcher(), 
											sessionId, 
											sessionPassword);
	
```



## 6.2 节点增删改查

创建节点有同步、异步两种形式，是重载的`create`方法：

1. 同步创建有返回值，成功返回节点路径，失败抛出异常`KeeperException`

2. 异步创建无返回值，成功调用参数中的回调方法`StringCallback.processResult()`，方法内容可以自己实现，也可以根据`ctx`执行不同的操作

3. 都不支持节点的递归创建

```java
// 同步创建，path,data,acl与命令create一致, createmode是-s序列 -e临时节点的结合体
public String create(final String path, byte data[], List<ACL> acl, CreateMode createMode) throws KeeperException, InterruptedException

// 异步创建，StringCallback是创建成功后的回调函数, ctx是成功后的返回信息,一般为json
public void create(final String path, byte data[], List<ACL> acl, CreateMode createMode,  StringCallback cb, Object ctx)
```

同步创建节点	

```java
	// 如果创建失败会抛出异常KeeperException
	@Test
	public void createNode() throws KeeperException, InterruptedException {
		/**
		 * 同步或者异步创建节点，都不支持子节点的递归创建，异步有一个callback函数
		 * 参数：
		 * path：创建的路径
		 * data：存储的数据的byte[]
		 * acl：控制权限策略
		 * 			Ids.OPEN_ACL_UNSAFE --> world:anyone:cdrwa
		 * 			CREATOR_ALL_ACL --> auth:user:password:cdrwa
		 * createMode：节点类型, 是一个枚举
		 * 			PERSISTENT：持久节点
		 * 			PERSISTENT_SEQUENTIAL：持久顺序节点
		 * 			EPHEMERAL：临时节点
		 * 			EPHEMERAL_SEQUENTIAL：临时顺序节点
		 */
		String result = zookeeper.create("/testnode", "123".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
		log.warn("创建" + result + "成功");
	}
```

异步创建节点	

```java
	@Test
	public void createNodeAsync() throws InterruptedException {
		String ctx = "{'create':'success'}";

		// 因为是异步,创建成功后调用StringCallback.processResult()
		zookeeper.create("/testnode3/abc", "123".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT, new AsyncCallback.StringCallback() {
			@Override
			public void processResult(int rc, String path, Object ctx, String name) {
				System.out.println("创建节点: " + path);
				System.out.println((String)ctx);
			}
		}, ctx);
		Thread.sleep(2000);
	}
```

设置节点数据

```java
	// 版本号错误会抛出KeeperException: Badversion for /node
	@Test
	public void setData() throws KeeperException, InterruptedException {
		/**
		 * 参数：
		 * path：节点路径
		 * data：数据
		 * version：数据版本
		 * 返回值Stat等价于stat命令, 返回节点状态信息
		 */
		Stat status = zookeeper.setData("/testnode", "666".getBytes(), 1);
		System.out.println(status.getVersion());
	}
```

删除节点

```java
	@Test
	public void deleteNodeAsync() throws KeeperException, InterruptedException {
		String result = "{'delete':'success'}";
		zookeeper.delete("/testnode", 2, new AsyncCallback.VoidCallback() {
			@Override
			public void processResult(int rc, String path, Object ctx) {
				System.out.println("删除节点" + path);
				System.out.println((String)ctx);
			}
		}, result);
	}
```

查询节点数据

```java
	/**
	 * 获取节点数据, 等价于命令 get path [watch]
	 * Stat保存节点状态信息, data保存节点数据
	 * watch=false表示不添加监听,为true表示添加监听,监听事件在watch的`process`中触发
	 */
	@Test
	public void getNodeData() throws KeeperException, InterruptedException {
		Stat status = new Stat();
		byte[] data = zookeeper.getData("/imooc", false, status);
		System.out.println("节点数据:" + new String(data));
	}
```

获取子节点列表

```java
	/**
	 * 获取子节点列表
	 * stat用于获得当前节点状态信息
	 */
	@Test
	public void getChildrenNode() throws KeeperException, InterruptedException {
		Stat status = new Stat();
		List<String> children = zookeeper.getChildren("/imooc", false, status);
		children.forEach(e -> System.out.println(e));
	}

```

判断节点是否存在

```java
	@Test
	public void nodeExist() throws KeeperException, InterruptedException {
		Stat status = zookeeper.exists("/imooc", false);
		if(status == null) {
			System.out.println("当前节点不存在");
		}else {
			System.out.println("当前节点存在，dataVersion：" + status.getVersion());
		}
	}
```



## 6.3 watch与acl





# 7 Apache Curator

Apache Curator也是一款开源的zookeeper客户端Java API，企业常用于操作zookeeper。API简单易用，提供常用的工具类，提供了**分布式锁解决方案**，并且解决了原生API的三个问题：

1. 超时重连，需要手动重连
2. watch注册后，一次触发就会失效
3. 不支持递归创建节点



## 7.1 使用Curator操作zk

[查看完整代码](github.com)

1. 引入依赖，`pom.xml`需要引入Curator依赖

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>4.0.0</version>
</dependency>

<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>4.0.0</version>
</dependency>
```

2. 创建客户端，设置重试策略

```java
	/**
    * 同步创建zk示例，原生api是异步的
    *
    * curator链接zookeeper的策略:ExponentialBackoffRetry
    * baseSleepTimeMs：初始sleep的时间
    * maxRetries：最大重试次数
    * maxSleepMs：最大重试时间
    */
	RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
	/**
	* 获取zk客户端，需要传入zk地址，超时时间，重试策略，命名空间
    * namespace,该客户端即所有增删改查操作的节点路径前面都会加上 /workspace
    */
    client = CuratorFrameworkFactory.builder()
        .connectString(zkServerPath)
        .sessionTimeoutMs(10000).retryPolicy(retryPolicy)
        .namespace("workspace").build();
    client.start();
```

3. 检查客户端连接状态，是否启动，测试关闭客户端

```java
    /**
     * 获取客户端的连接状态, 关闭会话连接
     * 用于替代过时方法isStarted()
     */
    @Test
    public void getzkStatus() throws InterruptedException {
        boolean isZkCuratorStarted = client.getState() == CuratorFrameworkState.STARTED;
        System.out.println("当前客户的状态：" + (isZkCuratorStarted ? "连接中" : "已关闭"));
        Thread.sleep(3000);

        client.close();
        boolean isZkCuratorStarted2 = client.getState() == CuratorFrameworkState.STARTED;
        System.out.println("当前客户的状态：" + (isZkCuratorStarted2 ? "连接中" : "已关闭"));
    }
```

4. 操作节点
   1. 创建节点、
   2. 删除节点、
   3. 设置节点数据、
   4. 获取节点数据和状态信息、
   5. 获取子节点列表，
   6. 判断节点是否存在

```java
    /**
     * 创建节点
     * <p>
     * client的命名空间是/workspace, 即所有增删改查的操作的节点路径前面都会加上 /workspace
     * 会在第一次创建节点时自动创建父节点/workspace
     * 如果节点已存在抛出异常KeeperException$NodeExistsException: KeeperErrorCode = NodeExists for /workspace/curator/imooc
     */
    @Test
    public void createNode() throws Exception {
        byte[] data = "abc".getBytes();
        String nodePath = "/curator/imooc";
        String path = client.create().creatingParentsIfNeeded()       // 如果父节点不存在,创建父节点
                .withMode(CreateMode.PERSISTENT)        // 设置节点-s -t 临时,序列界定啊
                .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)   // 设置用户控制权限
                .forPath(nodePath, data);

        System.out.println(path + "节点创建成功");
    }

    /**
     * 设置节点数据, 可设置dataVersion
     * @throws Exception
     */
    @Test
    public void setData() throws Exception {
        byte[] data = "123".getBytes();
        String nodePath = "/curator/imooc";

        Stat stat = client.setData()
//                .withVersion(2)   // 设置版本号,可省略, 若版本号错误抛出异常
                .forPath(nodePath, data);
        System.out.println("dataVersion" + stat.getVersion());
    }

    /**
     * 删除节点, 版本号可省略
     * 如果节点不存在会抛出异常KeeperException$NoNodeException: KeeperErrorCode = NoNode for /workspace/curator
     */
    @Test
    public void deleteNode() throws Exception {
        String nodePath = "/curator";
        client.delete()
                .guaranteed()       // 如果删除失败,那么后端会继续删除,直至成功
                .deletingChildrenIfNeeded()     // 如果存在子节点,就删
                //  .withVersion(2)
                .forPath(nodePath);
    }

    /**
     * 获取节点数据和状态信息
     * 状态信息保存在Stat中, 数据保存在data中
     *
     * @throws Exception
     */
    @Test
    public void getNode() throws Exception {
        String nodePath = "/curator/imooc";
        Stat stat = new Stat();

        byte[] data = client.getData()
                .storingStatIn(stat)    // 保存节点状态信息
                .forPath(nodePath);

        System.out.println(new String(data));
        System.out.println(stat.toString());
    }

    /**
     * 获取所有子节点名称
     *
     * @throws Exception
     */
    @Test
    public void getChildrenNode() throws Exception {
        List<String> nodes = client.getChildren().forPath("/curator");
        nodes.forEach((n) -> System.out.println(n));
    }

    /**
     * 判断节点是否存在
     */
    @Test
    public void nodeExist() throws Exception {
        Stat stat = client.checkExists().forPath("/aaa");
        if (stat == null) {
            System.out.println("节点不存在");
        } else {
            System.out.println("节点存在" + stat);
        }
    }
```

## 7.3 设置Watch事件

1. 设置**一次失效**的`watcher`事件

   ```java
       /**
        * 对节点设置watcher, 触发一次后失效
        */
       @Test
       public void setWatcher() throws Exception {
           CountDownLatch latch = new CountDownLatch(2);
           client.getData().usingWatcher(new CuratorWatcher() {
               @Override
               public void process(WatchedEvent watchedEvent) throws Exception {
                   System.out.println(watchedEvent.getPath() + "触发watcher事件: " + watchedEvent.getType());
   				// 只会执行一次
                   latch.countDown();
               }
           }).forPath("/curator/imooc");
   
           // 等待操作cmd客户端(set /workspace/curator/imooc 666), 触发监听器, 回调process方法
           // 修改节点数据, 触发一次后失效, 所以程序永远不会结束
           latch.await();
       }
   ```

2. **一次注册N次监听**的`wacher`事件，不区分watch事件类型，不监听子节点NodeChildrenChanged事件

```java
    /**
     * 利用nodeCache和Listener设置watch事件
     * 一次注册,N次监听
     * 缺点是多种类型Watch事件(NodeCreated, NodeDataChanged,NodeDeleted)都被称为NodeChanged, 但是不监听NodeChildrenChanged事件
     * @throws Exception
     */
    @Test
    public void setWatcherByNodeCache() throws Exception {
        // 这次的监听器一直有效, 所以设置为5
        CountDownLatch latch = new CountDownLatch(5);

        NodeCache nodeCache = new NodeCache(client, "/curator/imooc");
        nodeCache.start(true);      // true启动时缓存当前节点, false启动时不缓存节点

        if (nodeCache.getCurrentData() == null) {
            System.out.println("节点初始化数据为空");
        } else {
        	String data = new String(nodeCache.getCurrentData().getData());
            System.out.println("节点初始化数据为: " + data);
        }

        // 添加监听器, 等待节点被修改触发监听器,执行nodeChanged方法
        nodeCache.getListenable().addListener(new NodeCacheListener() {
            @Override
            public void nodeChanged() throws Exception {
                // 节点删除或不存在
                if(nodeCache.getCurrentData() == null) {
                    System.out.println("节点不存在");
                }
                // nodeCache.getCurrentData()是获取节点对象ChildData, 
                // ChildData可以获取节点路径,数据,状态信息stat
                String data = new String(nodeCache.getCurrentData().getData());
                System.out.println("节点" + nodeCache.getCurrentData().getPath() + " 数据为:" + data );

                latch.countDown();
            }
        });

        // 使主程序不结束, 等待cmd客户端修改节点触发监听器(set /workspace/curator/imooc  777)
        // 一次注册,N次监听, 修改节点数据5次, 触发10次watch事件后程序结束
        latch.await();
    }
```

7. **设置区分事件类型Watch事件**，一次注册，N次监听，区分事件类型。因为`PathChildrenCache`监听子节点，所以我们一般都设置为目标节点的父节点，然后在回调函数中筛选出目标节点。

```java
    /**
     * 监听节点, 需要异步初始化PathChildrenCache触发监听
     * 
     * @throws Exception
     */
    @Test
    public void setWatchsByPathChildrenCache() throws Exception {
        // 设置需要监听的节点
        String nodePath = "/curator/imooc";

        // PathChildrenCache是监听所有子节点, 所以设置为"/curator/imooc"的父节点/curator
        PathChildrenCache childCache = new PathChildrenCache(client, "/curator", true);
        /*
         * StartMode: 初始化方式
         * POST_INITIALIZED_EVENT：异步初始化，初始化之后会触发事件
         * NORMAL：异步初始化
         * BUILD_INITIAL_CACHE：同步初始化
         */
        childCache.start(PathChildrenCache.StartMode.POST_INITIALIZED_EVENT);

        // 注意这里获取的是子节点的数据,不是名称
        List<ChildData> childDataList = childCache.getCurrentData();
        for (ChildData data : childDataList) {
            System.out.println(new String(data.getData()));
        }

        childCache.getListenable().addListener(new PathChildrenCacheListener() {
            /**
             * @param curatorFramework 就是client, 可以根据监听事件操作节点, 比如监听到a节点修改了数据, 那b节点就删除client.delete().forPath("/b")
             * @param event  监听事件, 可以得到事件类型,节点名称,节点数据等
             */
            @Override
            public void childEvent(CuratorFramework curatorFramework, PathChildrenCacheEvent event) throws Exception {
                String path = event.getData().getPath();

                // 由于PathChildrenCache监听/curator的所有子节点,而我们只关心/curator/imooc, 所以使用卫语句进行排除
                if (!nodePath.equals(path)) {
                    return;
                }

                processEvent(event);
            }
        });
    }
	
	private void processEvent(PathChildrenCacheEvent event) {
        ChildData node = event.getData();
        switch (event.getType()) {
            case INITIALIZED:
                System.out.println("子节点初始化完成...");
                break;
            case CHILD_ADDED:   // 如果子节点已经创建, 则在启动时会触发该事件
                System.out.println("创建子节点:" + node.getPath());
                System.out.println("子节点数据:" + new String(node.getData()));
                break;
            case CHILD_UPDATED:
                System.out.println("修改子节点:" + node.getPath());
                System.out.println("修改子节点数据:" + new String(node.getData()));
                break;
            case CHILD_REMOVED:
                System.out.println("删除子节点:" + node.getPath());
                break;
            default:
                System.out.println("触发Watch事件,类型为:" + event.getType());
        }
    }
```



## 7.2 <a name="watchconfig">统一配置文件管理</a>

统一配置文件管理的原理是利用`watch`事件，比如为了同步redis配置文件到redis集群

1. 在zk上创建`redisConfig`节点
2. 所有redis集群机器上都启动zk的 Java **客户端**，并对`redisConfig`节点设置`watch`事件
3. 运维人员使用命令行修改redisConfig节点的数据，`set /workspace/conf/redis-config {"type":"update","url":"ftp://192.168.10.123/config/redis.xml"}`
4. 所有**zk客户端**监听到`DataChanged`事件，查看节点数据，解析数据后可知需要对redis配置文件的操作为`update`，配置文件地址为`url`。
5. 根据文件地址`url`下载redis配置文件，替换原有的配置文件，重启服务即可。

查看客户端代码

```java
/**
 * 统一配置文件管理的原理是利用watch事件，比如为了同步redis配置文件到redis集群
 *
 * 每台机器上都执行该类的main方法, 即启动zk客户端.
 */
public class Client1 {

    public static CuratorFramework client = null;
    public static final String zkServerPath = "localhost:2181";

    static {
        RetryPolicy retryPolicy = new RetryNTimes(3, 5000);
        client = CuratorFrameworkFactory.builder()
                .connectString(zkServerPath)
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy)
                .namespace("workspace").build();
        client.start();
    }

    public final static String CONFIG_NODE = "/conf/redis-config";
    public final static String CONFIG_NODE_PATH = "/conf";
    public static CountDownLatch countDown = new CountDownLatch(10);

    public static void main(String[] args) throws Exception {
        System.out.println("client1 启动成功...");

        final PathChildrenCache childrenCache = new PathChildrenCache(client, CONFIG_NODE_PATH, true);
        childrenCache.start(StartMode.BUILD_INITIAL_CACHE);

        // 添加监听事件
        childrenCache.getListenable().addListener(new PathChildrenCacheListener() {
            public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
                if (event.getData() == null) {
                    return;
                }

                String path = event.getData().getPath();
                // 只对/conf/redis-config节点的变化进行处理
                if (!CONFIG_NODE.equals(path)) {
                    return;
                }
                processEvent(event);
            }
        });

        countDown.await();

        client.close();
    }

    public static void processEvent(PathChildrenCacheEvent event) throws InterruptedException {
        // 只监听/redic-config节点变化的事件, 不监听创建、删除节点事件
        if (!PathChildrenCacheEvent.Type.CHILD_UPDATED.equals(event.getType())) {
            System.out.println(event.getType());
            return;
        }

        // 读取节点数据
        String jsonConfig = new String(event.getData().getData());
        System.out.println("节点" + CONFIG_NODE_PATH + "的数据为: " + jsonConfig);
        if (jsonConfig.isEmpty()) {
            System.out.println("配置文件json为空, 请重新输入");
        }
        JSONObject obj = JSON.parseObject(jsonConfig);//将json字符串转换为json对象
        String type = obj.getString("type");
        String url = obj.getString("url");
        // 判断操作类型,修改配置文件
        switch (type) {
            case "add":
                System.out.println("监听到新增的配置，文件路径为<"+ url + ">, 准备下载...");
                Thread.sleep(500);
                System.out.println("下载成功，已将配置文件添加到项目中");
                break;
            case "update":
                System.out.println("监听到新增的配置，文件路径为<"+ url + ">, 准备下载...");
                Thread.sleep(500);
                System.out.println("下载成功，已将配置文件替换到项目中");
                break;
            case "delete":
                System.out.println("监听到需要删除配置");
                Thread.sleep(100);
                System.out.println("成功删除项目中原配置文件");
                break;
            default:
                System.out.println("无法识别操作类型:" + type);
        }
        // TODO 视情况统一重启服务
    }
}
```

## 7.3 acl权限操作与认证授权

1. 创建节点时设置ACL权限

```java
 /**
     * 创建节点时设置acl权限
     * 创建成功后使用命令行查看节点权限
     * getAcl /workspace/curator/imooc/myacl
     * 'digest,'imooc1:ee8R/pr2P4sGnQYNGyw2M5S5IMU=
     * : cdrwa
     * 'digest,'imooc2:Ux2+KXVIAs1OI24TQ/0A9Yh0/QU=
     * : rw
     *
     * @throws Exception
     */
    @Test
    public void createAcl() throws Exception {
        String nodePath = "/curator/imooc/myacl";
        List<ACL> acls = new ArrayList<>();
        Id imooc1 = new Id("digest", getDigestUserPwd("imooc1:123456"));
        Id imooc2 = new Id("digest", getDigestUserPwd("imooc2:666666"));

        // 用户imooc1拥有所有权限, imooc2拥有读写权限
        acls.add(new ACL(ZooDefs.Perms.ALL, imooc1));
        acls.add(new ACL(ZooDefs.Perms.READ | ZooDefs.Perms.WRITE, imooc2));

        // 创建节点
        client.create()
                .creatingParentsIfNeeded()
                // 设置节点的acl权限, false表示不对父节点/curator/imooc/生效, 当父节点已存在时,true也不生效
                .withACL(acls, false)
                .forPath(nodePath, "123".getBytes());

    }

    public static String getDigestUserPwd(String id) {
        String digest = "";
        try {
            digest = DigestAuthenticationProvider.generateDigest(id);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        return digest;
    }
```

执行完上面的代码，创建节点成功后，使用命令行查看acl权限。

```bash
# imooc1拥有全部权限, imooc2拥有读写权限，与代码设置一直
[zk: localhost:2181(CONNECTED) 19] getAcl /workspace/curator/imooc/myacl
'digest,'imooc1:ee8R/pr2P4sGnQYNGyw2M5S5IMU=
: cdrwa
'digest,'imooc2:Ux2+KXVIAs1OI24TQ/0A9Yh0/QU=
: rw
# 不登录操作节点, 提示权限不合法
[zk: localhost:2181(CONNECTED) 20] set /workspace/curator/imooc/myacl 000
Authentication is not valid : /workspace/curator/imooc/myacl
```

2. 修改具有ACL权限控制节点的数据
   1. 使用用户登录并创建客户端
   2. 修改节点数据
   3. 重新设置节点ACL权限(需要用户具有admin权限)

```java
    /**
     * 获取权限限制的节点数据, 重新设置节点ACL
     * @throws Exception
     */
    @Test
    public void getDataAndSetAcl() throws Exception {
        String nodePath = "/curator/imooc/myacl";
        // 上个方法中设置了myacl的权限, client没有登录, 所以无法修改数据, 会抛出异常KeeperException$NoAuthException
        //client.setData().forPath(nodePath, "aaa".getBytes());

        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);

        // 使用用户imooc1登录客户端, imooc1具有管理权限
        CuratorFramework authClient = CuratorFrameworkFactory.builder()
                .connectString(zkServerPath)
                .authorization("digest", "imooc1:123456".getBytes())
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy)
                .namespace("workspace").build();
        authClient.start();

        authClient.setData().forPath(nodePath, "aaa".getBytes());

        /*
         * 修改数据后使用以下两个命令在Cli查看数据是否修改成功
         * addauth digest imooc2:666666
         * get /workspace/curator/imooc/myacl
         */

        // 设置节点的ACL权限,imooc2有了删除权限. 注意这里是重新设置, 而不是添加权限
        List<ACL> acls = new ArrayList<>();
        Id imooc1 = new Id("digest", getDigestUserPwd("imooc1:123456"));
        Id imooc2 = new Id("digest", getDigestUserPwd("imooc2:666666"));
        acls.add(new ACL(ZooDefs.Perms.ALL, imooc1));
        acls.add(new ACL(ZooDefs.Perms.READ | ZooDefs.Perms.WRITE | ZooDefs.Perms.DELETE, imooc2));

        authClient.setACL().withACL(acls).forPath(nodePath);

        // Cli使用 getAcl /workspace/curator/imooc/myacl  查看节点权限
    }
```



# 8 zookeeper 实现原理

## 8.1 为dubbo提供动态的服务注册和发现

**dubbo无法动态注册和发现**

比如项目中有多个订单服务，每个服务都是一台机器，每个客户端（这是Order请求的客户端，不是zk客户端）都有一份服务提供者列表。

高并发时需要添加多台机器或服务down掉了，服务的提供者发生了变化，结果客户端并不知道。

要想得到最新的服务提供者的URL列表，必须得手工更新配置文件才行，确实很不方便。

![20200303200126.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200303200126.png)

这就是客户端和服务提供者的紧耦合，想解除这个耦合，非得增加一个中间层不可。

**zookeeper注册中心**

所以应该有个注册中心，首先给这些服务命名（例如orderService），其次那些新增OrderService 都可以在这里注册一下，客户端就到这里来查询，只需要给出名称orderService，注册中心就可以给出一个可以使用的url， 再也不怕服务提供者的动态增减了。

![e1c0bc56c20024152140868979a5f98.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/e1c0bc56c20024152140868979a5f98.png)

zookeeper就可以充当上文中的注册中心，创建节点`/orderService`，提供订单服务的机器需要启动一个**zk客户端**，注册一个`node1`节点，节点数据保存服务的`url`

![20200303200412.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200303200412.png)
/orderService 表达了一个服务的概念， 下面的每个节点表示了一个服务的实例。 例如/orderService/node2表示的orderService 的第二个实例， 每个节点上可以记录下该实例的url , 这样就可以查询了。

当然这个注册中心必须得能和各个服务实例通信，如果某个服务实例不幸down掉了，那它在树结构中对于的节点也必须删除，这样客户端就查询不到了。

![20200303200444.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200303200444.png)
注册中心zookeeper就是和各个服务实例`node`之间建立Session，让各个服务实例的**zk客户端**定时发送心跳，如果过了特定时间收不到心跳，就认为这个服务实例node挂掉了，Session 过期， 把它从树形结构中删除。



## 8.2 用于实现分布式锁

同一个进程中，多个线程访问共享资源，可以使用Java提供的synchronized等锁就可以实现安全访问，但是在分布式系统中，程序都跑在不同机器的不同进程中，多个系统（进程）访问共享资源，就需要一个**分布式锁**

和synchronized一样，保证一个资源只能同时被一个节点抢到即可。谁能抢先在zookeeper创建一个`/distribute_lock`的节点就表示抢到这个锁了，然后读写资源，读写完以后就把`/distribute_lock`节点删除，其他进程再来抢。

这样存在一个缺点，某个系统可能会多次抢到，不太公平。

可以让这些系统在注册中心zookeeper的/distribute_lock下都创建**顺序节点**，会自动给每个节点一个编号，会是这个样子：

![20200303200531.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200303200531.png)
然后各个系统去检查自己的编号，谁的编号小就认为谁持有了锁， 例如系统1。

系统1持有了锁，就可以对共享资源进行操作了， 操作完成以后process_01这个节点删除， 再创建一个新的节点（编号变成process_04了）：

![20200303200547.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200303200547.png)
其他系统一看，编号为01的删除了，再看看谁是最小的吧，是process_02，那就认为系统2持有了锁，可以对共享资源操作了。 操作完成以后也要把process_02节点删除，创建新的节点。这时候process_03就是最小的了，可以持有锁了。

![20200303200640.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200303200640.png)

## 8.3 zookeeper 高可用

服务注册于发现和分布式锁的例子，都加入了一个中间层zookeeper，但是引入了一个重要的问题：

​	如果zookeeper挂掉，所有服务都依赖于zookeeper，那么就无法注册服务和发现服务，也无法获取分布式锁了，所以**必须保证注册中心zookeeper的高可用。**



为了实现高可用，zookeeper维护了一个集群，**一主多从**结构，如下图所示：

![20200303200709.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200303200709.png)

zookeeper会从Server集群选举出一个`Leader`节点（这里的节点是指服务器，不是 Znode），用于**接收写/读请求**。更新数据时，首先更新到`Leader`，再同步到`follwer`。

Server集群其他均为`follwer`，用于**接收读请求**，直接从当前follower Server读取。但是又出现了主从数据一致性问题。

> 如何保证zookeeper主从节点（Leader和follower）的数据一致性呢？

为了保证主从节点的数据一致性，ZooKeeper 采用了 **ZAB 协议(Zookeeper Atomic Broadcast)**，这种协议非常类似于一致性算法 Paxos 和 Raft。 ZAB 协议所定义的三种节点状态：

**Looking：**选举状态。

**Leading：**Leader 节点（主节点）所处状态。

**Following：**Follower 节点（从节点）所处的状态。

zk客户端会随机的链接到 zookeeper 集群中的一个`Leader`或`follower`节点，如果是读请求，就直接从当前节点中读取数据；如果是写请求，那么节点就会向 `Leader `提交事务，`Leader `接收到事务提交，会广播该事务，只要超过半数节点写入成功，该事务就会被提交，每一个事务都会使用`zxid`持久化到日志中，用于zk崩溃时恢复节点。

另外，Zookeeper是一个树形结构，具有顺序性很多操作都要先检查才能确定是否可以执行，比如P1的事务t1可能是创建节点"/a"，t2可能是创建节点"/a/b"，只有先创建了父节点"/a"，才能创建子节点"/a/b"。

为了实现这一点，Zab协议要保证同一个`Leader`发起的事务要按顺序被执行，同时还要保证只有先前`Leader`的事务被执行之后，新选举出来的`Leader`才能再次发起事务。



##8.4 Zookeeper 的崩溃恢复

> 如果主节点Leader宕机，那么如何恢复服务呢？

**1. 领导选举Leader election**

选举阶段，此时集群中的节点处于 Looking 状态。它们会各自向其他节点发起投票，投票当中包含自己的服务器 ID 和最新事务 ID（ZXID）。

![20200303200733.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200303200733.png)

接下来，节点会用自身的 ZXID 和从其他节点接收到的 ZXID 做比较，如果发现别人家的 ZXID 比自己大，也就是数据比自己新，那么就重新发起投票，投票给目前已知最大的 ZXID 所属节点。

![de93fb73e29d77d46f198ee82c51577.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/de93fb73e29d77d46f198ee82c51577.png)

每次投票后，服务器都会统计投票数量，判断是否有某个节点得到半数以上的投票。如果存在这样的节点，该节点将会成为准 Leader，状态变为 Leading。其他节点的状态变为 Following。

![20200303200809.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200303200809.png)
这就相当于，一群武林高手经过激烈的竞争，选出了武林盟主。

**2. 发现 Discovery**

发现阶段，用于在从节点中发现最新的 ZXID 和事务日志。或许有人会问：既然 Leader 被选为主节点，已经是集群里数据最新的了，为什么还要从节点中寻找最新事务呢？

这是为了防止某些意外情况，比如因网络原因在上一阶段产生多个 Leader 的情况。

所以这一阶段，Leader 集思广益，接收所有 Follower 发来各自的最新 epoch 值。Leader 从中选出最大的 epoch，基于此值加 1，生成新的 epoch 分发给各个 Follower。

各个 Follower 收到全新的 epoch 后，返回 ACK 给 Leader，带上各自最大的 ZXID 和历史事务日志。Leader 选出最大的 ZXID，并更新自身历史日志。

**3. 同步 Synchronization**

同步阶段，把 Leader 刚才收集得到的最新历史事务日志，同步给集群中所有的 Follower。只有当半数 Follower 同步成功，这个准 Leader 才能成为正式的 Leader。

自此，故障恢复正式完成。



## 8.5 zookeeper 数据写入过程

写入数据就涉及到了 ZAB协议的 **BroadCast (广播)阶段**，简单来说，就是 Zookeeper 常规情况下更新数据的时候，由 Leader 广播到所有的 Follower。详细过程如下：

1. zk客户端发出写入数据请求给任意Follower。

2. Follower 把写入数据请求转发给 Leader。

3. Leader 采用二阶段提交方式，先发送广播给 Follower。

4. Follower 接到 Propose 消息，写入日志成功后，返回 ACK 消息给 Leader。

5. Leader 接到半数以上 ACK 消息，返回成功给客户端，并且广播 Commit 请求给 Follower。

![20200303200856.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200303200856.png)
# 9 zookeeper 分布式锁

## 9.1 Curator与Spring的结合
见参考文档2

## 9.2 什么是分布式锁



## 9.2 实现分布式锁



# 分布式一致性算法

集群中有两个数据库A和B，为了保证一致性，所以A和B需要同步数据。当User更新了数据库A的数据value后，User从数据库B读取数据value，此时会出现三种情况：

1. **强一致性**，value==2。强一致性需要让同步过程非常快（很难实现）；或者利用**分布式锁**，在读取数据库B前阻塞住，等待同步完成后释放锁
2. **弱一致性**，value==1 。数据更新后，如果能容忍后续的访问只能访问到部分或者全部访问不到，则是弱一致性。最终一致性就属于弱一致性。
3. **最终一致性**，最终value==2。一段时间后，节点间的数据会最终达到一致状态，但不保证在任意时刻任意节点上的同一份数据都是相同的

![一致性.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/一致性.png)

更多一致性问题参考文章[强一致性、顺序一致性、弱一致性和共识](https://blog.csdn.net/chao2016/article/details/81149674)。










# 待补充

后面根据极客时间《zookeeper实战与源码解析》（8小时视频）补充笔记，包括

1. 实现服务发现，
2. 解析paxos和raft，对比Chubby，使用etcd，
3. 存储结构，存储源码，
4. 客户端服务端通信源码，
5. 节点选举，ZAB

根据博客等逐步更新一下内容：

1. CAP理论  
2. 服务端同步原理，
3. 客户端响应原理，
4. 可视化客户端工具ZooInspector和exhibitor 
5. zookeeper异步初始化的源码分析，eventThread，sendThread

**总结:** 
啥都不懂公众号, 观其大略有印象; 
快速入门看周阳, 短小生动门槛低; 
想要开发看慕课, 制作精良能实战; 
深入理解看极客, 大牛源码说原理.  
依次耗时更长，学习曲线更陡峭，但是也更深入

# 推荐阅读

1. [什么是zookeeper - 码农翻身](https://mp.weixin.qq.com/s/J8erMBhiogXoQcn91SJcbw)，讲了zookeeper诞生是为了解决哪些问题，即zk的作用
2. [分布式一致性算法 - 码农翻身](https://mp.weixin.qq.com/s/ohTXhFFywGHGDOkzO45aaQ)
3. [强一致性、顺序一致性、弱一致性和共识](https://blog.csdn.net/chao2016/article/details/81149674)
4. [什么是zookeeper - 程序员小灰](https://mp.weixin.qq.com/s/Gs4rrF8wwRzF6EvyrF_o4A)
5. [如何用zookeeper实现分布式锁 - 程序员小灰](https://mp.weixin.qq.com/s/u8QDlrDj3Rl1YjY4TyKMCA)
6. [zookeeper 面试题 - 附答案](<https://mp.weixin.qq.com/s/VbVlYeZ8n0hOV7PpA4sLCw>)，用于检查学习成果和复习
7. [观察者模式](https://www.bilibili.com/video/av57936239?p=117)，zookeeper是一个基于观察者模式设计的分布式服务管理框架


# 参考文档

1. [ZooKeeper分布式专题与Dubbo微服务入门 - 慕课网](https://coding.imooc.com/class/201.html)
2. [zookeeper 代码仓库 - github](https://github.com/maoturing/imooc-zookeeper)
3. [深入浅出理解Zookeeper - 周阳](pan.baidu.com)
4. [Zookeeper实战与源码剖析 - 极客时间](https://time.geekbang.org/course/intro/100034201)
5. [zookeeper源码解读](https://b23.tv/av82609500)
6. [什么是zookeeper - 码农翻身](https://mp.weixin.qq.com/s/J8erMBhiogXoQcn91SJcbw)
7. [分布式一致性算法 - 码农翻身](https://mp.weixin.qq.com/s/ohTXhFFywGHGDOkzO45aaQ)
8. [强一致性、顺序一致性、弱一致性和共识](https://blog.csdn.net/chao2016/article/details/81149674)
9. [什么是zookeeper - 程序员小灰](https://mp.weixin.qq.com/s/Gs4rrF8wwRzF6EvyrF_o4A)
10. [如何用zookeeper实现分布式锁 - 程序员小灰](https://mp.weixin.qq.com/s/u8QDlrDj3Rl1YjY4TyKMCA)
11. [Java中的乐观锁](https://www.cnblogs.com/mmmmar/p/8624242.html)     
12. [zookeeper 面试题 - 附答案](<https://mp.weixin.qq.com/s/VbVlYeZ8n0hOV7PpA4sLCw>)
13. [Java 异步实现的几种方式](https://blog.csdn.net/weixin_43525116/article/details/85698872)


# 错误总结

1. [ZooKeeper 启动报错java.lang.NumberFormatException](https://blog.csdn.net/HeatDeath/article/details/79174164?utm_source=blogxgwz2)

2. [Curator NodeCache的错误使用](https://www.jianshu.com/p/43348aa23e99)

