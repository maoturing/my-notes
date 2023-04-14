> 问题

1. 生产环境发生了内存溢出该如何处理？
2. 生产环境应该给服务器分配多少内存？
3. 如何对垃圾收集器的性能进行调优？
4. 生产环境CPU负载飙升如何处理？
5. 生产环境应该给应用分配多少线程？
6. 如何不加log就确定是否执行了某一行代码？
7. 如何不加log就能实时查看某个方法的参数值和返回值?
8. JVM的字节码
9. 循环体中用"+"做字符串拼接为什么效率低，“+”的底层一定是StringBuilder.append()吗？
10. String常量池
11. i++与++i哪种写法效率更高？
12. jvm监控和调试工具的使用
13. 理解JVM的GC机制，学会GC调优
14. TomCat性能监控与调优

# 一  基于JDK命令行工具的监控

## 1. JVM的三种参数类型

### 1.1 标准参数

  jvm的标准参数，在jvm各个版本中基本不变。
  使用`java`获取所有标准参数

- -help
- -server -client ^①^
- -version -showversion
- -cp -classpath

使用`java -version`命令可以输出java的版本信息，从第4行可以看到：

  JVM的**名字**(HotSpot)、**类型**(Server)、**build ID**(24.79-b02)，JVM以**混合模式**(mixed mode)在运行，这是HotSpot默认的运行模式，意味着JVM在运行时可以动态的把字节码编译为本地代码。关于切换类型（Server Client）以及二者的区别详见参考文档1。

```shell
C:\Users\Administrator>java -version
java version "1.8.0_51"
Java(TM) SE Runtime Environment (build 1.8.0_51-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.51-b03, mixed mode)
```

  

### 1.2 X 参数

  非标准化参数，在jvm各个版本中会有一些小变化。
  使用`java -X`获取所有X参数

- -Xint：JVM以解释方式执行所有的字节码，会显著降低运行速度（interpreted）

- -Xcomp：JVM在第一次使用时就把所有字节码编译为本地代码（compiled）

- -Xmixed：混合模式**(默认)**，由jvm自己决定是否编译为本地代码，会将字节码中多次被调用的部分便以为本地代码以提高执行效率；被调用很少的方法会在解释模式下执行，减少编译和优化成本 

  ```shell
  # 默认是mixed mode(混合模式),在java的版本信息中显示
  C:\Users\Administrator>java -version
  java version "1.8.0_51"
  Java(TM) SE Runtime Environment (build 1.8.0_51-b16)
  Java HotSpot(TM) 64-Bit Server VM (build 25.51-b03, mixed mode)
  
  # 修改为interpreted mode(解释执行)
  C:\Users\Administrator>java -Xint -version
  java version "1.8.0_51"
  Java(TM) SE Runtime Environment (build 1.8.0_51-b16)
  Java HotSpot(TM) 64-Bit Server VM (build 25.51-b03, interpreted mode)
  
  # 修改为compiled mode(编译执行)
  C:\Users\Administrator>java -Xcomp -version
  java version "1.8.0_51"
  Java(TM) SE Runtime Environment (build 1.8.0_51-b16)
  Java HotSpot(TM) 64-Bit Server VM (build 25.51-b03, compiled mode
  ```

  测试三种JVM工作模式可以使用死循环代码测试运行时间，具体命令如下：

  ```shell
  javac HelloWorld.java
  java -Xint HelloWorld
  java -Xcomp HelloWorld
  java -Xmixed HelloWorld
  ```

### 1.3 XX 参数

      非标准化参数，在jvm各个版本中变化较大，主要用于jvm调优和debug，也是**最常用**的参数
      使用`java -XX:+PrintFlagsFinal -version > flags1.txt`命令获取所有XX参数^③^，大约有700+参数

- Boolean 类型

  格式：-XX:[+-]<name>  表示启用或者禁用name属性

  举例：-XX:+UseConcMarkSweepGC   启用CMS垃圾收集器
             -XX:+UseG1GC	 启用G1垃圾收集器

- K-V 类型

  格式：-XX:[+-]<name>=<value>    表示name属性的值是value

  举例：-XX:InitialHeapSize=3116367872    初始化堆内存，常用`-Xms512m`表示，不要误认为是X参数

  |                                                              |
  | ------------------------------------------------------------ |
  | `-XX:MaxHeapSize=3116367872`   最大堆内存，常用`-Xmx1024m`表示 |
  | `-XX:ThreadStackSize=1024`    线程堆栈大小，常用`-Xss128k`表示 |
  | `-XX:MaxGCPauseMills=500 `   表示GC的最大停顿时间是500ms     |
  | `-XX:GCTimeRatio=19`                                         |

### 1.4 常用命令     

- `jps`：查看java进程。

  `-m`显示传递给`main`方法的参数。

  `-l`显示应用程序`main`类的完整包名称或应用程序的JAR文件的完整路径名。

  `-v`显示传递给JVM的参数        

- `jinfo`：查看Java进程的修改过的jvm参数

  `jinfo -flags 2345`查看进程id为2345的jvm参数

  `jinfo -flag MaxheapSize 2345`查看进程id为2345的jvm参数最大堆内存`MaxheapSize `的值

## 2. jstat查看虚拟机统计信息

###2.1 类加载信息

```shell
# 进程id21640 输出间隔1000ms 输出次数3
# Loaded加载类的数量  Bytes加载的kB数  Unloaded：卸载的类数  Bytes：卸载的Kbytes数Time：执行类加载和卸载操作所花费的时间
C:\Users\Administrator>jstat -class 21640 1000 3
Loaded  Bytes  Unloaded  Bytes     Time
  3010  5823.5       52    77.1       3.48
  3010  5823.5       52    77.1       3.48
  3010  5823.5       52    77.1       3.48
```

###2.2 垃圾回收信息

```
# 输出结果各项指标详见参考文档2
C:\Users\Administrator>jstat -gc 21640
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
18432.0 19456.0  0.0    0.0   77824.0  48459.9   126976.0   10715.1   19456.0 18288.3 2304.0 1968.8     13    0.204   1      0.139    0.342
```
这里利用springboot来创建一个Controller，模拟gc
```java
@Controller("/bug")
public class MemController {
	public static final int m = 1024 * 1024;

	@RequestMapping(method = RequestMethod.GET)
	@ResponseBody
	public String allocationMem(Integer num) {
		System.out.println("request success.....");
		for (int i = 0; i < num; i++) {
			byte[] mem = new byte[m];
		}
		return "success";
	}
}
```
启动项目之后，我们访问http://localhost:8080/bug?num=100，就可以增加内存了，使用`jstat -gcutil`命令查看内存占用，可以发现，访问前Eden区内存占比为55.43%，访问后32.46%，中间发生了一次YGC
```
C:\Users\User>jstat -gcutil 7264 200 3
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00   0.00  55.43  18.17  95.65  93.81      5    0.050     2    0.089    0.139
  0.00   0.00  55.43  18.17  95.65  93.81      5    0.050     2    0.089    0.139
  0.00   0.00  55.43  18.17  95.65  93.81      5    0.050     2    0.089    0.139

C:\Users\User>jstat -gcutil 7264 200 3
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  7.57   0.00  32.46  18.17  95.24  93.35      6    0.053     2    0.089    0.141
  7.57   0.00  32.46  18.17  95.24  93.35      6    0.053     2    0.089    0.141
  7.57   0.00  32.46  18.17  95.24  93.35      6    0.053     2    0.089    0.141
```

### 2.3 JIT编译信息

```shell
# 查看JIT编译信息
C:\Users\Administrator>jstat -compiler 21640
Compiled Failed Invalid   Time   FailedType FailedMethod
    3054      1       0    12.35          1 org/apache/tomcat/util/IntrospectionUtils setProperty
```



## 3. 分析内存溢出 [实战]

![堆内存结构](http://upload-images.jianshu.io/upload_images/3274507-0306e979e52c1c03.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 分析内存溢出分两步：
1. 打印内存dump，见3.2节。包括`jmap`命令，`jcmd [pid] GC.heap_dump filepath`命令，arthas中的`heapdump filepath`等
2. 分析dump文件，找出导致内存溢出的对象，见3.3节。包括 jvisualvm，Jprofile 等


### 3.1 模拟内存溢出

```java
/**
 * Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit 
 * exceeded
 */
private static void heapSize2() {
    Map<Object,Object> map = new HashMap();
    Random r = new Random();
    Integer i = 0;
    while (true) {
        map.put(i++, "aaa");
    }
}

/**
 * Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
 */
private static void heapSize() {
    List<String> names = new ArrayList();
    for (;;) {
        names.add("bbb");
    }
}
```

### 3.2 生成dump文件

1. 使用命令行参数，内存溢出时自动导出**（最常用）**

   `-XX+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./`

   另外也可以使用`-Xrunhprof:head=site` 参数生成java.hprof.txt 文件，不过这样会影响 JVM的运行效率，不建议在生产环境中使用

2. 使用`jmap`命令手动导出   `jmap -dump:format=b,file=heap.hprof 2345`

    `format=b`表示以二进制形式导出，`file=heap.hprof`表示文件名称为heap.hprof，`2345`表示要操作的java进程id
    更多`jmap`命令选项详见参考文档2

3. 使用JConsole生成Dump文件![JConsole生成dump文件](http://upload-images.jianshu.io/upload_images/3274507-f5b7aca795d7b755.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4. 使用`jcmd`命令生成Dump文件，参考文章 https://www.jianshu.com/p/bc2ff6829d2d

### 3.3 使用 jvisualvm 分析dump文件

使用 jvisualvm 导入分析内存 dump 文件，详细参考文章 https://www.jianshu.com/p/065d12dd3e44

![使用jvisualvm分析内存dump](https://upload-images.jianshu.io/upload_images/3274507-25921632412cdfcc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


相比MAT更加简洁清晰，而且 jvisualvm 为JDK自带工具，不需要额外下载。

### 3.3 使用MAT分析dump文件
导出dump文件之后，就是分析dump文件了，这里我们选择最常用的MAT来进行分析。首先[下载](https://www.eclipse.org/mat/downloads.php)MAT (Memory Analyzer)，主要参考[MAT教程](https://www.javatang.com/archives/2017/10/30/53562102.html)^④⑤^，这里我们只介绍一些常用的功能。

![mat分析](http://upload-images.jianshu.io/upload_images/3274507-1c4e26b14c6e66a6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Shallow Heap表示对象本身所占的内存^④⑤^，Retained Heap表示对象和对象中的引用总共占的内存大小。比如User对象包含name属性，会引用String对象。



排除虚引用，根据GC Roots查看对象引用![mat根据GC Roots查看对象引用](http://upload-images.jianshu.io/upload_images/3274507-88c2522996a2333d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



![mat查看对象引用树](http://upload-images.jianshu.io/upload_images/3274507-1b7c927b8b38935c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 4. 分析死循环与死锁 [实战]

> 分析CPU负载过高总共分两步
1. 使用`top`命令找到CPU负载最高的进程，使用`top -Hp [pid]`找到CPU负载最高的进程
2. 打印线程堆栈信息，查看该线程堆栈，分析可能出错的代码。打印堆栈信息可以用`jstack`命令，`jcmd [pid] Thread.print -> filepath`命令，arthas 可以省略第 1 步直接打印CPU负载最高的 n 个线程`thread -n 3`


### 4.1 模拟 CPU 飙升
代码见[Github](https://github.com/vipcolud/monitor/blob/master/src/main/java/com/imooc/monitor_tuning/chapter2/CpuController.java)

### 4.2 查找高负载线程和打印线程堆栈

CPU负载(load average)^⑦^和使用率过高，很可能的原因就是死循环。

1. 首先可以使用top命令查看占用CPU最高的**进程信息**，获取进程pid

2. 使用Linux命令`top -p [pid] -H`监控指定进程中所有**线程信息**，然后找到CPU占用率高的线程nid，`-p`参数表示指定 pid，`-H`参数显示线程的信息，包括线程 ID 和 CPU 占用率等
（windows中可以process exlporer工具查看指定进程中所有**线程信息**）

   ![top打印指定进程的所有线程信息](http://upload-images.jianshu.io/upload_images/3274507-c01c8ab3cd4d6c98.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   如图可以看到`8247(0x2037)`线程的CPU使用率达到了96.7%，此时我们就可以确认是这个线程的问题了。(4核心电脑，每个线程最大CPU使用率为100%，总使用率最大为400%)

3. 使用`jstack [pid] > myStack.txt`命令打印指定java进程的所有线程堆栈信息

4. 在`myStack.txt`文件中查看线程id为**nid**的堆栈信息（`myStack.txt`线程id是**16进制**，`top -p`命令线程id是**10进制**），找到导致CPU飙升的`线程[10进制 PID=8247，16进制 nid=0x2037]`堆栈信息了，然后分析问题，解决问题。![jstack查看线程信息](http://upload-images.jianshu.io/upload_images/3274507-91fdd30e873d91d5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据上图中`线程8247(0x2037)`的堆栈信息，我们猜测是`CpuController.loop()`方法导致的cpu负载过高，然后我们就可以分析代码，解决问题了。

5. 另外`jstack`命令还能自动**检测死锁**，如果存在死锁，在myStack.txt末尾会有一行`found 1 deadlock`，也会打印死锁线程的相关信息，根据这些信息就可以轻松定位到死锁代码，解决CPU负载过高的问题了。

![jstack死锁线程信息](http://upload-images.jianshu.io/upload_images/3274507-50fd6f4c6d5a6d8d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

另外，相比`jstack`命令来检测死锁，我们还有更轻松的方式，使用JConsole工具来检测死锁。
![使用JConsole检测死锁](https://upload-images.jianshu.io/upload_images/3274507-82cce72ee424d906.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



// 未完待续

# 二 基于JVisualVM的可视化监控



# 三  基于Btrace的监控调试

## 4.1 简介

BTrace可以动态地向目标应用程序的字节码注入追踪代码，用到的技术JavaCompilerAPI，JVMTI，Agent，Instrumentation+ASM

1、接口性能变慢，分析每个方法的耗时情况； 

2、当在Map中插入大量数据，分析其扩容情况； 

3、分析哪个方法调用了System.gc()，调用栈如何； 

4、执行某个方法抛出异常时，分析运行时参数；

##4.2 环境准备

下载JVisualVM BTrace插件https://visualvm.github.io/pluginscenters.html

下载BTrace https://github.com/btraceio/btrace/releases/tag/v1.3.11

jar包

## 4.2 使用BTrace

###4.3.1 简单使用

```java
import com.sun.btrace.AnyType;
import com.sun.btrace.BTraceUtils;
import com.sun.btrace.annotations.BTrace;
import com.sun.btrace.annotations.Kind;
import com.sun.btrace.annotations.Location;
import com.sun.btrace.annotations.OnMethod;
import com.sun.btrace.annotations.ProbeClassName;
import com.sun.btrace.annotations.ProbeMethodName;

@BTrace
public class PrintArgSimple {
	// 在Ch4Controller类的fun1方法的入口处(ENTRY)进行追踪
	@OnMethod(clazz = "com.imooc.monitor_tuning.chapter4.Ch4Controller", 
			method = "fun1", location = @Location(Kind.ENTRY))
	public static void anyRead(@ProbeClassName String className, @ProbeMethodName String methodName, AnyType[] args) {
		BTraceUtils.printArray(args);  //fun1方法的所有参数
		BTraceUtils.println(className + ":" + methodName);
	}
}
```

`btrace [pid] MyBtrace.java`

### 4.3.2 拦截构造函数，重载方法

### 4.3.3 拦截返回值，异常，行号

### 4.3.4 拦截复杂参数，环境变量

### 4.3.5 注意事项



# 四 Tomcat性能监控与调优

# 五  JVM层GC调优

当项目较大且启动较慢时（如Kettle），需要加载的类较多，占用元空间较大，我们可以打印GC日志
-XX:+PrintGCDetails  -XX:+PrintGCTimeStamps  -XX:+PrintGCDateStamps -Xloggc:./gc.log

下图是项目启动的GC日志，7s时间就出现了两次`Full GC(Metadata GC threshold)`，说明元空间较小，我们可以在启动参数中使用命令
-XX:MetaspaceSize
-XX:MaxMetaspaceSize
来调节元空间大小，提高项目的启动速度 
![image.png](https://upload-images.jianshu.io/upload_images/3274507-e410cf48f86c7922.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 六 JVM字节码与Java代码层调优

# 七 总结





![image](http://upload-images.jianshu.io/upload_images/3274507-5def9e82f103a229.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



参考文档:

1. [关于JVM的类型和模式](http://www.importnew.com/20715.html)
2. [Java命令行工具帮助文档 - Oracle](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/index.html)
3. [打印所有XX参数](http://wiki.jikexueyuan.com/project/jvm-parameter/all.html)
https://www.jianshu.com/p/bc2ff6829d2d
4. [生成dump文件与MAT的使用](https://www.javatang.com/archives/2017/10/30/53562102.html)
5. [MAT使用教程](http://wensong.iteye.com/blog/1986449)
6. [MAT中文文档](https://download.csdn.net/download/mayle/10131356)
7. [理解Linux系统负荷 - 阮一峰](http://www.ruanyifeng.com/blog/2011/07/linux_load_average_explained.html)
8. [JVisual VM使用教程](https://www.cnblogs.com/kongzhongqijing/articles/3625340.html)
9. [如何在生产环境使用Btrace进行调试 - 占小狼](https://www.jianshu.com/p/dbb3a8b5c92f)
10. [如何回答"线上CPU100%排查"面试问题](https://mp.weixin.qq.com/s/MpFrTvKg8Gde5FrhmcfePQ)
11. [jcmd + jvisualvm 进行内存泄漏及OOM异常分析](https://www.jianshu.com/p/065d12dd3e44)


> 扩展阅读

[如何使用MAT进行内存泄露分析 - 占小狼](https://www.jianshu.com/p/738b4f3bc44b)
[JVM系列](https://www.cnblogs.com/redcreen/tag/jvm/)
[使用 JITWatch 查看 JVM 的 JIT 编译代码](http://www.importnew.com/28957.html)
[深入浅出 JIT 编译器](https://www.ibm.com/developerworks/cn/java/j-lo-just-in-time/index.html)
[JVM杂谈之JIT](https://zhuanlan.zhihu.com/p/28476709)
[关于JVM内存的N个问题](http://www.cnblogs.com/QG-whz/p/9636366.html#commentform)
[jvisualvm安装Visual GC插件](https://blog.csdn.net/shuai825644975/article/details/78970371)
[MAT入门到精通(一) - dqVoic](https://mp.weixin.qq.com/s/eNk1Nesy4Tsb0hO-9Rv9vw)
[JVM发生OOM的 8 种原因、及解决办法 - 占小狼](https://mp.weixin.qq.com/s/HwsU282ZuXUFjq4Oi7Br_A)
[JVM 性能调优监控工具 jps、jstack、jmap、jhat、jstat、hprof 使用详解](https://mp.weixin.qq.com/s/XBB2IJf8ODkcjZiU423J4Q)
[jcmd 使用教程](https://www.jianshu.com/p/bc2ff6829d2d)
[arthas 视频教程](https://www.bilibili.com/video/BV18v41167o1)
[arthas 官方中文教程](https://arthas.aliyun.com/doc/dashboard.html)