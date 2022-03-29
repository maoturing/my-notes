[TOC]



# 0 前言

> 为什么需要学习并发编程?
1. 大厂JD硬性要求，也是高级工程师必经之路，几乎所有的程序都需要并发和多线程
2. 面试高频出现，书籍、网络博客内容水平参差不齐，知识点凌乱
3. 众多框架的原理和基础，Spring 线程池、单例的应用；数据库的乐观锁思想；Log4J2 对阻塞队列的应用

本门课程的优点
----------
1. 系统：成体系不容易忘记，思维导图，为什么->演示代码->分析原理->得出结论
2. 内容丰富：线程 8 大核心基础，java内存模型，死锁
3. 分析面试题：答题思路，引申解答
4. 分析本质：深入原理分析设计理念，interrupt与stop停止线程，wait必须在同步块中使用，JMM
5. 学习方法：技术提高途径、技术前沿动态、业务中成长、自顶向下学习
6. 通俗易懂：近朱者赤-happen-before，森然火灾-线上事故，夫妻迁让-死锁
7. 逐步迭代：从0开始，逐渐优化，重视思路，分析错误代码到修复问题
8. 案例演示丰富
9. 习题检验：总结知识点，检验学习效果，知识卡防止走神
10. 配套资料：思维导图，知识点文档，面试题总结

线程八大核心
-----
![20191227022954.png](https://i.loli.net/2019/12/27/6qsZr4e7QU31pD2.png)

# 1. 创建多线程（核心1）

查看[Oracle Java官方文档](https://docs.oracle.com/javase/8/docs/api/) 或 `Thread` 类注释，可知**创建线程有两种方法，即实现`Runable`接口和继承`Thread`类**。
```java
/**
73 * There are two ways to create a new thread of execution. One is to declare a class to be a subclass of <code>Thread</code>

99 * The other way to create a thread is to declare a class that implements the <code>Runnable</code> interface.
**/
```
**1. 实现`Runable`接口**
```java
public class RunnableStyle implements Runnable {
    public static void main(String[] args) {
        // 将将我们创建的 RunnableStyle 作为构造函数参数传入Thread
        Thread t = new Thread(new RunnableStyle());
        t.start();
    }

    @Override
    public void run() {
        System.out.println("实现Runnable接口创建线程");
    }
}
```

**2. 继承`Thread`类**
```java
public class ThreadStyle extends Thread {
    public static void main(String[] args) {
        ThreadStyle t = new ThreadStyle();
        t.start();
    }
    @Override
    public void run() {
        System.out.println("继承Thread类创建线程");
    }
}
```
两种创建方式的对比
---------
**方法1 实现 Runable 接口更好**：
1. 可扩展，java 只能单继承多实现，继承 Thread 类后就不能继承其他类，限制了可扩展性；而 Runnable 方式可以实现多个接口
2. 节约资源，继承 Thread 类每次要新建一个任务，每次只能去新建一个线程，而新建一个线程开销是比较大的（具体在第8章），需要创建、执行和销毁；而 Runnable 方式可以利用线程池工具，可以避免创建线程、销毁线程带来的开销。线程创建需要开辟虚拟机栈、本地方法栈、程序计数器等线程私有的内存空间，线程销毁时需要回收这些资源，频繁创建销毁线程会浪费大量系统资源
3. 解耦，实现 Runnable 接口解耦，一是具体的业务逻辑在`run()`方法中，二是控制线程生命周期是 Thread 类，两个目的不一样，不建议写在一个类中，应该解耦。？


**本质区别**

方式1 实现 Runable 创建线程，查看下方 `Thread.run()`代码和注释可知，启动线程前，要将我们创建的 RunnableStyle 作为 Thread 构造函数的参数`target`，所以实际运行的是`target.run()`，即我们线程类 RunnableStyle 的`run()`方法

方式2 继承Thread类，会重写`Thread.run()`方法，启动线程后直接运行我们线程类 ThreadStyle 的`run()`方法

```java
public class Thread implements Runnable {
    
    private Runnable target;
    
    // 构造函数，传入我们写的Runnable
    public Thread(Runnable target) {...}

    /**
     * If this thread was constructed using a separate
     * <code>Runnable</code> run object, then that
     * <code>Runnable</code> object's <code>run</code> method is called;
     * otherwise, this method does nothing and returns.
     * <p>
     * Subclasses of <code>Thread</code> should override this method.
     */
    @Override
    public void run() {
        if (target != null) {
            // 调用Runnable的run()方法
            target.run();
        }
    }
}
```
> **思考题：** 同时使用 Runnable 和 Thread 两种创建线程的方式会怎么样？

 Runnable 方式中我们创建的 Runnable 实例 target 会作为构造函数参数传入到Thread类，然后被`Thread.run()`调用执行，而继承 Thread 方式会覆写`Thread.run()`方法，就使得 target 不会被调用执行，[查看详细代码](github)

 > **面试题 1：** 创建/实现线程有几种方式？

 1. 有**两种方法**，根据 Thread 类的注释（或 Java 官方文档），分别是实现 Runnable 接口和继承 Thread 类
 2. 准确的讲，本质都是**一种方式**，本质都是构造 Thread 类，调用`Thread.run()`方法，只不过一种 Runnable 实现类作为 target 传入Thread，然后调用`target.run()`方法，另一种是直接重写 run() 方法（参考上面本质区别）
 3. 分析优缺点，**可扩展性、节约资源、解耦**（参考上面两种创建方式的对比）
 4. 分析常见的 6 种典型错误观点，线程池、Callable等**本质都是实现 Runnable 接口**

6 种典型错误观点分析
-----

- 线程池创建线程也算一种新建线程的方式  [示例代码](github)

ExecutorService 本质都是使用线程工厂创建线程，查看源码可知线程工厂都是 **实现 Runnable 接口**构造 Thread 类的方法创建线程

```java
    // @see java.util.concurrent.Executors.DefaultThreadFactory#newThread(java.lang.Runnable)
    
    // 线程池的线程工厂，创建线程的方式如下，实现Runnable接口，构造Thread类
    public Thread newThread(Runnable r) {
        // 传入用户的Runnable实例，设置线程组，线程名称等
        Thread t = new Thread(group, r,
                                namePrefix + threadNumber.getAndIncrement(),
                                0);
        // ......
        return t;
    }
```
创建线程池也可以用户自定义线程工厂，从下方代码中也可以看出，用户自定义线程工厂线程工厂也是**实现 Runnable 接口**构造 Thread 类的方法创建线程
```java
// 用户自定义线程工厂，见码出高效p239
public class UserThreadFactory implements ThreadFactory {
    private final String namePrefix;
    private final AtomicInteger nextId = new AtomicInteger(1);

    UserThreadFactory(String whatFeatrueOfGroup) {
        namePrefix = "UserThreadFactory's " + whatFeatrueOfGroup + "-Worker-";
    }

    @Override
    public Thread newThread(Runnable task) {
        String name = namePrefix + nextId.getAndIncrement();

        // task是用户实现 Runnable 接口创建的，构造Thread类创建线程
        Thread thread = new Thread(null, task, name, 0);
        System.out.println(thread.getName());
        return thread;
    }
}
```
-  通过 Callable 和 FutureTask 创建线程，也算是一种新建线程的方式 [示例代码](CallableStyle)

**查看[示例代码](CallableStyle)可知**，Thread构造函数参数是 futureTask，Runnable 的实现类，查看下方`FutureTask`代码，可知`FutureTask.run()`调用了`Callable.call()`方法，并把返回值保存到`FutureTask.outcome`，**本质还是实现 Runnable 接口。**
```java
public class FutureTask implements RunnableFuture {

    private Object outcome;     // 保存call()返回值
    private Callable callable;

    // 构造方法，出入用户创建的callable
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }

    @Override 
    public void run() {
        // .....
        // callable是FutureTask构造函数传入的
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 在run()方法中调用用户定义的Callable.call()
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);    // 将返回值保存到outcome
        }
        // ......
    }
}
```

![20191227052650.png](https://i.loli.net/2019/12/27/UYJ51EmQkWflb7q.png)

- 无返回值是实现 Runnable 接口，有返回值是实现 Callable 接口，所以 Callable 是新的创建线程的方式 [示例代码]() 
  

与上一个问题类似，本质还是要借助 FutureTask（**FutureTask 实现了 Runnable 接口**），构造 Thread 类创建线程，启动线程后会调用`target.run()`，即`FutureTask.run()`，其中会调用`Callable.call()`方法，所以不算是一种新的创建线程方式，**本质还是实现 Runable 接口**，不过是创建 FutureTask 实现 Runnable 接口的工作JDK帮我们做了。

- 定时器


- 匿名内部类 [示例代码]()

**运行[示例代码]()**，会生成AnonymousInnerClassStyle$1.class，反编译发现该类**实现了 Runnable 接口**，AnonymousInnerClassStyle$2.class反编译发现该类**继承了 Thread 类**
- lambda 表达式 [示例代码]()

**查看[示例代码]()** ，代码中打印了 lambda 表达式实现的接口，创建线程本质还是**实现 Runnable 接口**，但并不完全等价于匿名内部类的方式，逻辑与下方代码类似。

虽然也是匿名内部类，不会生成内部类的 .class 文件，而会**动态生成**内部类`LambdaStyle$$Lambda$1`，lambda 表达式中的内容会被编译成静态方法`LambdaStyle.lambda$main$0()`，**动态生成**的内部类 Runnable 实例`LambdaStyle$$Lambda$1`直接调用静态方法`LambdaStyle.lambda$main$0()`，详细参考《Java8 实战》附录D和掘金小册[《JVM 字节码从入门到精通》第9节](https://juejin.im/book/5c25811a6fb9a049ec6b23ee/section/5cb52721f265da039b08629e)
```java
public class LambdaStyle {

    private static void lambda$main$0() {
        System.out.println("hello, lambda");
    }
}

final class LambdaStyle$$Lambda$1 implements Runnable {
    @Override
    public void run() {
        LambdaStyle.lambda$main$0();
    }
}
```
> **面试题 2**：实现 Runnable 接口和继承 Thread 类的哪种方式更好？

1. 可扩展性，Java 不支持多继承，继承 Thread 类后就不能继承其他类，限制了可扩展性，而实现 Runnable 方式可以实现多个接口
2. 代码架构角度，实现 Runnable 接口解耦，一是具体的业务逻辑在`run()`方法中，二是控制线程生命周期是 Thread 类，两个目的不一样，不建议写在一个类中，应该解耦。
3. 节约资源，继承 Thread 类，新建任务只能去 new 一个对象，但是资源损耗比较大，继承 Thread 类每次要新建一个任务，每次只能去新建一个线程，而新建一个线程开销是比较大的（具体在第8章），需要创建、执行和销毁；而 Runnable 方式可以利用线程池工具传入Runnable 实例 target，可以避免创建线程、销毁线程带来的开销。线程创建需要开辟虚拟机栈、本地方法栈、程序计数器等线程私有的内存空间，线程销毁时需要回收这些资源，频繁创建销毁线程会浪费大量系统资源


彩蛋：学习编程知识的优质路径
------
- 宏观

    1. **责任心**，不要放过任何 Bug，找到原因并去解决，因为很多 Bug 都需要非常深入的知识才能解决，解决问题的能力比学很多的知识更重要
    2. **主动**，永远不要觉得自己的时间多余，不断重构、优化、学习、总结
    3. **敢于承担**，对于没碰过的技术难题，在一定调研后，敢于承担，让工作充满挑战，攻克难关的过程进步飞速
    4. **关心产品和业务**，不仅要写好代码，更要在业务层面多思考
- 微观

    1. **系统学习**，碎片化知识公众号文章最容易忘记和一叶障目，要看经典书籍的译本
    2. **官方文档**，专家撰写不断迭代，最权威的，像线程实现方式百度结果有非常多的错误
    3. **分析源码**，随着看源码和官方文档的次数增多，就会更熟悉更快，而baidu并不能达到这个效果
    4. **英文搜索**，前面几个不能解决问题，再搜索 Google 和 StackOverflow，用英文更容易找到正确答案如Annoymouses Class
    5. **多实践**，遇到新知识，多动手写Demo，并尝试用到项目里，三个阶段，看、写、生产环境

> 彩蛋：如何了解技术领域的最新动态

- 高质量固定途径，掘金、阮一峰博客
- 订阅技术论坛，InfoQ
- 公众号

> 彩蛋：如何在业务开发中成长

- 偏业务方向开发，了解业务核心模型架构，如电商交易、订单、结算等核心系统的设计
- 偏技术方向开发，**通用性非常强**，就业方向广，如中间件、RPC，APM
- **两个 25% 理论**，在一个领域达到前 25% 比较容易，但前 5% 很难，如果能在两个领域做到前 25%，一旦把两个领域能结合起来，就能做到非常优秀的 5%，如小灰是编程+写作领域，liuyubo是编程+授课领域，雷军是编程+管理，两个领域 25% 非常不错的职业规划


# 2. 启动多线程（核心2）

查看[示例代码]()，启动线程调用`start()`方法，然后会调用本地方法`start0()`，开辟新线程，而调用`run()`方法相当于Main线程调用方法，无法启动新线程

Thread 类的源代码分析
------

```java
    /* Java thread status for tools, initialized to indicate thread 'not yet started'
     * 线程的状态标志，初始化为0来标志线程尚未start()
     */
    private volatile int threadStatus = 0;
    
    // synchronized保证线程安全，即同一个线程对象不能同时调用start()方法
    public synchronized void start() {
        /**
         * 0 状态值对应线程的 NEW 状态，如果两次调用start()方法会抛出线程状态异常
         * A zero status value corresponds to state "NEW".
         * 
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        // 添加到线程组
        group.add(this);

        boolean started = false;
        try {
            // 调用native方法start0
            start0();
            started = true;
        } finally {
            // ......
        }
    }

    // native方法，开辟新线程，更改线程状态threadStatus，C++代码
    private native void start0();
```
`start0()`方法在 [`Thread.c`](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/b860bcc84d51/src/share/native/java/lang/Thread.c) 中，具体逻辑在[`jvm.cpp`](http://hg.openjdk.java.net/jdk7/jdk7/hotspot/file/tip/src/share/vm/prims/jvm.cpp)，详细见 [Java 线程源码解析之 start](https://www.jianshu.com/p/81a56497e073)

--------------

> 面试题 3：一个线程能调用两次`start()`方法吗？会发生什么情况？

1. 一个线程只能调用一次`start()`方法，否则会抛出IllegalThreadStateException，因为`start()`方法的卫语句会检查线程状态，线程状态不为 NEW 则抛出异常
2. `start()` 是 `synchronized` 修饰的**线程安全方法**，不存在线程已启动但线程状态 threadStatus 还未修改的情况，所以也不用担心被调用两次
3. 即使线程执行结束（TERMINATED）也不能再调用`start`方法，只有线程状态为 NEW 时才可以调用 start。**线程池复用的线程是不退出的**，复用的是Runnable实例，而不是对同一个线程调用了多次`start()`方法，见[第4章](#4-线程状态)
4. 为什么这么设计？

**问题？** 线程结束后能不能再调用start，那什么时候结束？

> 面试题 4：既然`start()`还是会调用`run()`方法，为什么我们不直接调用`run()`方法呢？

1. 因为调用`start()`方法才会真的启动一个新线程，而调用`run()`只是简单的调用方法
2. `start()`方法会调用本地方法`start0()`，然后开辟新线程，代码可以在 OpenJDK 中查看（详细参考 第2章 启动多线程）



# 3. 停止线程（核心3）

一般情况下都是线程执行完毕之后停止，如果用户想要主动停止线程，可以使用`Thread.interrupt`来通知线程停止，`Thread.interrupt`并不能真正的中断线程，而是「通知线程应该停止了」，具体到底停止还是继续运行，应该由**被通知的线程自己写代码处理**。


具体来说，当对一个线程，调用 interrupt() 时：
1. 如果线程处于**正常活动状态**，那么会将该线程的中断标志位设置为 true，仅此而已
2. 如果线程处于**被阻塞状态**（例如处于sleep, wait, join 等状态），那么线程将立即退出被阻塞状态，并抛出一个InterruptedException异常。仅此而已。
3. 被设置中断标志的线程将继续正常运行，不受影响，具体停止线程的逻辑需要自己写代码处理



停止正常活动状态线程
---------
停止正常活动线程的代码逻辑是Main线程使用`interrupt()`方法发送中断请求（相当于修改了标志位），当任务线程接收到中断请求，然后使用`Thread.interrupted()`检测标志位，退出while循环，结束线程，代码范式如下：

```java
    Thread thread = new Thread(() -> {
        // 检测中断标志位，如果收到中断请求，退出while循环。所有的业务逻辑应该都写在while循环中
        while (!Thread.interrupted()) {
            // do more work.
        }
    });
    thread.start();

    // 一段时间以后
    thread.interrupt();
```
```java
public class RightWayStopThreadWithoutSleep {
    public static void main(String[] args) throws InterruptedException {
        Runnable r = () -> {
            // 当接收到中断信号，退出循环，任务结束
            while (!Thread.interrupted()) {
                System.out.println("线程正在运行..");
            }
        };

        Thread t = new Thread(r);
        t.start();
        Thread.sleep(100L);    // 等待线程启动完成

        System.out.println("是否收到中断信号：" + t.isInterrupted());
        /*
         * 发送中断信号,改变中断标志位, 仅此而已
         * 如果线程循环条件是while (true), t线程会继续执行下去
         * 如果线程循环条件是while (!Thread.interrupted()), t线程会退出
         */
        t.interrupt();
        System.out.println("是否收到中断信号：" + t.isInterrupted());
    }
}
```
停止被阻塞状态线程
-----

代码逻辑任务线程处于被阻塞状态（例如处于·`sleep`, `wait`, `join`等状态），Main线程使用`interrupt()`方法发送中断通知改变中断标志位，**当任务线程调用`sleep`等方法时，发现中断标志位被修改，Java虚拟机会先将该线程的中断标志位复位，然后立即退出被阻塞状态**，并抛出一个`InterruptedException`异常 

```java
public class RightWayStopThreadWithSleepEveryLoop {
    public static void main(String[] args) throws InterruptedException {
        Runnable r = () -> {
            int num = 0;
            try {
                // 每次循环都会sleep的，不需要!Thread.currentThread().isInterrupted()判断条件，
                // 因为在抛出InterruptedException之前Java虚拟机会先将该线程的中断标志位复位，
                // 即使调用!Thread.currentThread().isInterrupted()返回也是true
                while (num <= 10000) { 
                    if (num % 100 == 0) {
                        // 验证JVM是否将中断标志位复位，返回true，说明复位了，故加在while循环条件中无用
                        System.out.println(!Thread.currentThread().isInterrupted());    // 
                        System.out.println(num + "是100的倍数");
                    }
                    num++;

                    // 这里会检测中断标志位，如果被修改则抛出异常退出while循环
                    Thread.sleep(10);
                }
            } catch (InterruptedException e) {
                System.out.println("收到中断信号Interrupt, 抛出异常, 结束线程");
                e.printStackTrace();
            }
        };
        Thread thread = new Thread(r);
        thread.start();
        // 等待线程完全启动
        Thread.sleep(5000);
        // 发送中断通知，修改中断标志位
        thread.interrupt();
    }
}

```
通过上面的代码我们也可以清楚的知道，这也是在调用`Thread.sleep()`方法时需要处理`InterruptedException`异常的原因


不能停止的线程
----
运行下面的示例代码，可以发现与上一小节的运行结果不同，会一直进行循环，**原因是`try-catch`没有包住`while`循环**，线程在`sleep`阻塞状态时，收到`interrupt`信号，抛出异常，然后被`catch`住，无法退出`while`循环结束线程，所以代码会一直运行直到`num <= 10000`
```java
public class CantInterrupt {
    public static void main(String[] args) throws InterruptedException {
        Runnable runnable = () -> {
            int num = 0;
            while (num <= 10000 && !Thread.currentThread().isInterrupted()) {
                if (num % 100 == 0) {
                    System.out.println(num + "是100的倍数");
                }
                num++;
                
                // 收到interrupt`信号，复原中断标志位，抛出异常，但是无法退出while循环，所以会继续运行
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        Thread thread = new Thread(runnable);
        thread.start();
        // 等待线程完全启动
        Thread.sleep(5000);
        // 发送中断通知
        thread.interrupt();
    }
}
```


`Thread.interrupted()`与`Thread.currentThread().isInterrupted()`的区别
------
查看源码易知，都是返回当前线程的中断标志位，`Thread.interrupted()`会复原标志位，`Thread.currentThread().isInterrupted()`不会
```java
    /** 
     * Tests whether the current thread has been interrupted.  The
     * <i>interrupted status</i> of the thread is cleared by this method. 
     * Thread.interrupted()方法检测当前线程是否已经中断，这个方法会将中断标志位interrupted status清除复位
     * 
     * In other words, if this method were to be called twice in succession, the second call would return false
     * 换句话说，如果该方法被调用两次，第二次会返回false
     */
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
    /**
     * Thread.currentThread().isInterrupted()返回线程的中断标志位，
     * 与上面代码的区别是参数ClearInterrupted为false，即不清除标志位
     */
    public boolean isInterrupted() {
        return isInterrupted(false);
    }
```
停止线程的最佳实践
---
由于`Runnable.run()`方法签名不允许抛出异常，所以只能catch住，所以需要传递中断。总之，无论如何，都不应屏蔽中断

为什么不扩大try-catch范围 ？
```java
public class RightWayStopThreadInProd implements Runnable {

    @Override
    public void run() {
        while (true && !Thread.currentThread().isInterrupted()) {
            System.out.println("...");
            try {
                throwInMethod();
            } catch (InterruptedException e) {
                // 阻塞状态受到中断信号，jvm会复位中断标志位，
                // 这里设置中断标志位为false，用于传递中断，使得while条件可以结束线程
                Thread.currentThread().interrupt();
                //保存日志、停止程序
                System.out.println("保存日志");
                e.printStackTrace();
            }
        }
    }

    // 业务方法
    private void throwInMethod() throws InterruptedException {
            Thread.sleep(2000);
    }

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new RightWayStopThreadInProd());
        thread.start();
        Thread.sleep(1000);
        thread.interrupt();
    }
}
```
上面代码依赖sleep检查中断标志位，如果没有调用sleep，应该怎么写？没有sleep更简单，直接while条件判断中断标志位即可

响应中断的方法列表
----
响应中断的意思是这这些方法的执行中，如果中断信号过来了，是可以感知到的。我们可以使用下面的方法让线程进入阻塞状态，为了使线程从阻塞状态恢复，就可以使用`interrupt()`方法中断线程
```java
Object.wait()/wait(long)/wait(long, int)
Thread.sleep()/sleep(long)/sleep(long, int)
Thread.join()/join(long)/join(long, int)
java.util.concurrent.BlockingQueue.take()/put(E)
java.util.concurrent.locks.Lock.lockInterruptibly()
java.util.concurrent.CountDownLatch.await()
java.util.concurrent.CyclicBarrier.await()
java.util.concurrent.Exchanger.exchange(V)
java.nio.channels.InterruptibleChannel
java.nio.channels.Selector
```
> 为什么要使用`interrupt`来停止线程，有什么好处？

被中断的线程有如何响应中断的权利，因为线程的某些代码可能是非常重要的，我们必须要等待线程处理完后，再由线程自己主动去中止，或者线程不想中止也是可以的，不应该鲁莽的使用`stop`，而应该使用`interrupt`方法发送中断信号，这样使得线程代码更加安全，数据的完整性也得到了保障。


错误的线程停止方法
---
~~`stop()`~~：过期方法，悟空说会导致数据不完整，但是加synchronized可以解决此问题，具体原因不知

~~`suspend()`~~：

~~`resume()`~~：

用`volatile`设置标记位：

彩蛋：如何分析 native 方法
---

1. 查看`Thread.interrupt()`源码，发现底层是调用`native`方法`private native void interrupt0();`
```java
    public void interrupt() {
        // ... 
        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();       // 调用native方法interrupt0
    }
```

2. 进入Github查看[OpenJDK代码库](https://github.com/AdoptOpenJDK/openjdk-jdk8u)，点击<kbd>FindFile</kbd>搜索`Thread.c`文件，JDK native 方法的源码都在同类名的 .c 文件中
![20200107061222.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200107061222.png)


3. 找到native方法对应的方法名`JVM_IsInterrupted`，在本仓库中搜索方法名`JVM_IsInterrupted`，发现在`jvm.cpp`中定义了该方法
```c
// Thread.c文件中可以知道Native方法interrupt0对应的本地方法是JVM_Interrupt
static JNINativeMethod methods[] = {
    {"interrupt0",       "()V",        (void *)&JVM_Interrupt},
};
```
```
    {"isInterrupted",    "(Z)Z",       (void *)&JVM_IsInterrupted},

JVM_ENTRY(void, JVM_Interrupt(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_Interrupt");     // 绑定对应方法

  // Ensure that the C++ Thread and OSThread structures aren't freed before we operate
  oop java_thread = JNIHandles::resolve_non_null(jthread);
  MutexLockerEx ml(thread->threadObj() == java_thread ? NULL : Threads_lock);
  // We need to re-resolve the java_thread, since a GC might have happened during the
  // acquire of the lock
  JavaThread* thr = java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread));
  if (thr != NULL) {
    Thread::interrupt(thr);
  }
```
4. 上面代码调用了`Thread::interrupt(thr)`，我们查看`thread.cpp`源码，找到该方法
```c++
void Thread::interrupt(Thread* thread) {
  trace("interrupt", thread);
  debug_only(check_for_dangling_thread_pointer(thread);)
  os::interrupt(thread);
}
```
5. 找到os::interrupt(thread)源码在os_windows.cpp中，然后分析源码
```c++
void os::interrupt(Thread* thread) {
  assert(!thread->is_Java_thread() || Thread::current() == thread || Threads_lock->owned_by_self(),
         "possibility of dangling Thread pointer");

  OSThread* osthread = thread->osthread();
  osthread->set_interrupted(true);
  // More than one thread can get here with the same value of osthread,
  // resulting in multiple notifications.  We do, however, want the store
  // to interrupted() to be visible to other threads before we post
  // the interrupt event.
  OrderAccess::release();
  SetEvent(osthread->interrupt_event());
  // For JSR166:  unpark after setting status
  if (thread->is_Java_thread())
    ((JavaThread*)thread)->parker()->unpark();

  ParkEvent * ev = thread->_ParkEvent ;
  if (ev != NULL) ev->unpark() ;

}
```

> **面试题 5**：如何停止一个线程？

使用interrupt发送中断通知，


> **面试题 6**：如何处理不可中断的阻塞？

# 4. 线程状态（核心4）
`Thread`的内部类`State`源码如下
```java
public enum State {
    // 更多线程状态的描述信息见源码注释
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
}
```

线程一共有 6 种状态，Java线程在运行的生命周期中会处于下表所示的6种不同的状态，同一时刻，线程只能处于其中的一个状态
| 状态名称     | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| NEW          | 新建状态，线程被创建，但还没有调用start()方法               |
| RUNNABLE     | （可）运行状态，Java线程将操作系统中的就绪Ready和运行Running两种状态合称为“可运行Runnable状态”，此状态的线程可能正在执行，也有可能正在等待CPU为它分配执行时间 |
| BLOCKED      | 阻塞状态，表示线程在等待着一个排他锁，                                  |
| WAITING      | 等待状态，表示当前线程需要等待其他线程显式唤醒或中断，这种状态的线程不会被CPU分配执行时间。wait()，join()等方法会让线程进入无限期的等待状态                       |
| TIME_WAITING | 计时等待状态，表示当前线程需要等待其他线程唤醒或中断，但是需要设置最长等待时间，等待超时会进入RUNNABLE状态，这种状态的线程也不会被CPU分配执行时间                                               |
| TERMINATED   | 终止状态，表示当前线程已经执行完毕                           |

线程 6 种状态的转换图如下所示，可以知道，
1. `start0()`会将线程状态从 NEW 修改到 RUNNABLE，Debug 观察`this.getState()`可知 
2. NEW、RUNNABLE、TERMINATED三种状态只能从前往后，不可逆，
3. BLOCKED、WAITING、TIME_WAITING三种状态都可以与RUNNABLE相互转换。

当线程状态到达TERMINATED，如果还想执行任务，需要重新创建线程，复用Runnable实现类即可

![sadfq1qwrewq.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/sadfq1qwrewq.png)


![20191229065809.png](https://i.loli.net/2019/12/29/qk25XtUBNYPa7p8.png)


阻塞状态
----
一般习惯而言，把BLOCKED（被阻塞），WAITING（等待），TIME_WAITING（计时等待）都称为阻塞状态
> **面试题 7**：线程的生命周期是什么，线程有哪几种状态？
根据上面的线程生命周期图进行描述，线程有 6 种状态，转换关系和转换条件。

> **面试题 7**：为什么Java线程没有Running状态？
https://mp.weixin.qq.com/s/c0VplSy83Ck6Wee54mK_ew

# 5. 线程的方法（核心5）
Object.wait()  释放调用对象的锁，进入`WAITING`状态

Object.notify()  唤醒同一对象一个`WAITING/TIMED_WAITING`状态的线程，用户无法指定具体唤醒的线程

Object.notifyAll()   唤醒同一对象所有`WAITING/TIMED_WAITING`状态的线程，

Thread.sleep() 进入`WAITING`状态，不释放锁，因为不释放锁，所以`sleep`都是有参方法，需要设置时间，否则会持有锁永久等待

## 5.1 wait/notify 实现线程通信

线程 t1 调用了`object.wait()`进入等待状态，线程 t2 调用`object.notify()/notifyAll()`唤醒 t1。

需要注意的是`object.notify()/notifyAll()`唤醒的是调用了`object.wait()`的线程，**需要保证是同一个对象，并且先`wait`后`notify`**。

**前提：** 由同一个lock对象调用wait、notify方法。

1. 当线程A执行wait方法时，该线程会被挂起
2. 当线程B执行notify方法时，会唤醒一个被挂起的线程A

> 面试题：**lock对象、线程A和线程B三者是一种什么关系？**

根据上面的结论，可以想象一个场景：

1. lock对象维护了一个等待队列list
2. 线程A中执行lock的wait方法，把线程A保存到list中
3. 线程B中执行lock的notify方法，从等待队列中取出线程A继续执行


```java
public class Wait {
    public static Object object = new Object();

    public static void main(String[] args) throws InterruptedException {
        Runnable r = () -> {
            // 获取object的monitor锁
            synchronized (object) {
                System.out.println("线程" + Thread.currentThread().getName() +"开始执行了");
                try {
                    // 调用wait，释放object的monitor锁
                    // wait方法必须在synchronized中调用
                    object.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("线程" + Thread.currentThread().getName() + "获取到了锁");
            }
        };

        Runnable r2 = () -> {
            // 线程1释放锁后，进入同步块
            synchronized (object) {
                // 唤醒线程1，执行完毕后，线程1开始执行
                object.notify();
                System.out.println("线程" + Thread.currentThread().getName() + "调用了notify");
            }
        };
        Thread t1 = new Thread(r, "t1");
        Thread t2 = new Thread(r2, "t2");
        t1.start();
        // 等待线程1启动, 这样才能保证先wait后notify
        Thread.sleep(200);    
        t2.start();
    }
}
```


wait()、notify()、notifyAll()方法需要在`synchronized`块中调用，即必须获取对象Monitor锁，否则会抛出`IllegalMonitorStateException`异常
```java
// wait方法必须在synchronized块中调用，否则会报异常IllegalMonitorStateException
public class WaitException {
    public static Object object = new Object();

    public static void main(String[] args) {

        Thread t = new Thread(() -> {
            try {
                object.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("...");
        });

        // 启动线程，会抛出IllegalMonitorStateException
        t.start();
    }
}
```
> **面试题 6**：如何处理不可中断的阻塞？

## 5.2 生产者消费者模型

[查看实例代码](https://github.com/maoturing/java_concurrency_core/blob/master/src/threadcoreknowledge/threadobjectmethods/ProducerConsumerModel.java)可知：
1. 生产者消费者模型有三个组成部分：**仓库、生产者、消费者**

2. 仓库有一个属性容量`maxsize`，两个功能生产`put`和消费`take`

3. `put`和`take`必须是线程安全的，防止多个生产者生产产品数量超出仓库`maxsize`

4. 生产者`put`时当仓库满了进入等待状态（调用wait），不再生产，等待消费者消费并唤醒自己；消费者`take`时仓库空了进入等待状态，不再消费，等待生产者生产后并唤醒自己



> **面试题 7**：两个线程交替打印 0-100 的奇偶数，即 A 线程只打印奇数，B 线程只打印偶数



思路1：synchronized关键字，缺点是奇数线程释放锁后并不一定是偶数线程拿到锁，会多次进入无用循环，性能较差
```java
public class PrintOddEvenSync {
    private static Object lock = new Object();
    private static int count = 0;

    public static void main(String[] args) {
        new Thread(() -> {
           while (count < 100) {
               synchronized (lock) {
                   if((count & 1) == 0) {
                       System.out.println(Thread.currentThread().getName() + ": " + count);
                       count++;
                   }
               }
           }
        }, "偶数Even").start();

        // 奇数线程
        new Thread(() -> {
            while (count < 100) {
                synchronized (lock) {
                    if((count & 1) == 1) {
                        System.out.println(Thread.currentThread().getName() + ": " + count);
                        count++;
                    }
                }
            }
        }, "奇数Odd").start();
    }
}
```
思路2：wait/notify，线程A打印偶数数字之后，唤醒另一个线程，自己进入等待状态；线程B打印奇数数字之后，唤醒另一个线程，自己进入等待状态。

A线程打印完后唤醒了其他线程，还未进入状态，此时CPU切换到了线程B，打印数字，唤醒其他线程，但是此时A还没有进入`WAITING`状态，就会导致永久等待。所以需要synchronized保证同一时刻只有一个线程在打印，当然，wait/notify也只能在同步方法中调用

```java
public class PrintOddEvenWait {
    private static Object lock = new Object();
    private static int count = 0;

    public static void main(String[] args) throws InterruptedException {
        Runnable r = () -> {
            while (count < 100) {
                synchronized (lock) {
                    System.out.println(Thread.currentThread().getName() + ": " + count);
                    count++;
                    // 打印之后，唤醒其他线程
                    lock.notify();
                    if(count < 100) {
                        try {
                            // 进入等待状态
                            lock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        };

        new Thread(r, "偶数").start();
        // 使用sleep保证偶数线程先启动，或者使用CountDownLatch
        Thread.sleep(100);
        new Thread(r, "奇数").start();
    }
}
```
> **面试题 8**：手写生产者消费者设计模式？

见上面 生产者消费者模型，[查看示例代码](https://github.com/maoturing/java_concurrency_core/blob/master/src/threadcoreknowledge/threadobjectmethods/ProducerConsumerModel.java)

> **面试题 9**：wait和sleep有什么区别，为什么 wait 需要在同步代码块中使用，而 sleep 不需要？

1. 区别见 **wait 和 sleep 的区别**
2. wait释放锁的前提是获取了对象的独占锁Monitor，调用wait()之后，当前线程又立即释放掉锁，线程随后进入WAIT_SET（等待池）中。正如wait方法的注释所说：This method should only be called by a thread that is the owner of this object's monitor
3. wait 在执行之后需要其他线程去 notify 唤醒，但是 **wait 不一定能保证在 notify 之前执行**（**线程切换执行**），如果 notify 先执行，wait 后执行就不能释放锁，可能会导致**永久等待或死锁**，所以在 synchronized 同步代码块中使用是为了保证 wait/notify的先后顺序

> **面试题 10**：为什么线程通信方法 wait/notify/notifyAll 被定义在Object中，而sleep定义在Thread类中？
wait/notify/notifyAll 属于锁操作，而锁状态标志保存在对象头中`Mark Word`，所以应该定义在Object中。

sleep 是线程操作，所以定义在 Thread 类中。

> **面试题 11**：wait 方法是属于 Object 的，那调用 Thread.wait 会怎么样？


> **面试题 12**：如何选择 notify 和 notifyAll？

唤醒一个线程还是唤醒全部线程，notify 无法指定唤醒的线程

> **面试题 13**：notifyAll 会唤醒所有线程，但是只有一个线程能获取到锁，那其他线程怎么办？

其他线程会进入 BLOCKED 阻塞状态，这类似于起初多个线程获取锁，获取不到的线程会进入 BLOCKED 状态等待锁释放

> **面试题 13**：可以用suspend和resume来阻塞线程吗？

这两个方法已经过时了，推荐使用wait/和notify来阻塞唤醒线程

sleep 方法
---
**作用：** 我只想让线程在预期时间执行，其他时候不要占用 CPU 资源；例如定时检查等

下面演示了sleep不释放锁
```java
public class SleepDontReleaseLock implements Runnable {
    private static final Lock lock = new ReentrantLock();

    public static void main(String[] args) {
        SleepDontReleaseLock sleepDontReleaseLock = new SleepDontReleaseLock();
        new Thread(sleepDontReleaseLock).start();
        new Thread(sleepDontReleaseLock).start();
    }

    @Override
    public void run() {
        lock.lock();
        System.out.println("线程" + Thread.currentThread().getName() + "获取到了锁");
        try {
            Thread.sleep(5000);
            System.out.println("线程" + Thread.currentThread().getName() + "已经苏醒");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            // 一定要解锁
            lock.unlock();
        }
    }
}
```

sleep 的优雅写法
---
```java
// 休眠3小时25分1秒，休眠时间小于0直接忽略，而sleep会抛出异常
TimeUnit.HOURS.sleep(3);
TimeUnit.MINUTES.sleep(25);
TimeUnit.SECONDS.sleep(1);
```

wait 和 sleep 的区别
----
- 相同
1. wait 和 sleep 都可以使线程进入（广义）阻塞状态
2. wait 和 sleep 都是可中断方法，被中断后会抛出InterruptException

- 不同
1. wait 是 Object 的方法，而 sleep 是 Thread 特有的方法
2. wait 方法的调用必须在同步方法中进行，而 sleep 不需要
3. 线程在同步方法中执行 wait 时会释放 monitor 锁，而 sleep 并不会释放 monitor 锁 
4. sleep 方法短暂休眠后会主动退出阻塞，而 wait（没有指定时间）则需要等待其他线程中断或唤醒

join 方法
----
**作用：** 因为新的线程加入了，我们需要等待他执行完成，**如果线程 M 中执行了`t1.join()`方法，表示当前线程 M 等待线程 t1 执行完毕后才开始执行线程 M**，[查看示例代码](https://github.com/maoturing/java_concurrency_core/blob/master/src/threadcoreknowledge/threadobjectmethods/Join.java)


**注意：** Main 线程中调用子线程的`join`方法，表示 Main 线程需要等待子线程执行完毕，**Main 线程会进入`WAITING`状态，而非子线程**，这点与`sleep`，`wait`不同，验证方法见[示例代码](https://github.com/maoturing/java_concurrency_core/blob/master/src/threadcoreknowledge/threadobjectmethods/JoinThreadState.java)。有些资料说`join`会进入`BLOCKED`状态是错误的

因为进入`WAITING`状态的是Main线程，所以中断`WAITING`状态需要在子线程中调用`main.interrupt()`，具体见[示例代码](https://github.com/maoturing/java_concurrency_core/blob/master/src/threadcoreknowledge/threadobjectmethods/JoinInterrupt.java)

**原理：** 查看下面`Thread`类的源码，可知`join()`方法底层调用的是`wait()`，由于**每个线程执行完后都会唤醒等待在该线程对象上的其他线程**（源码在 [Thread.cpp](https://github.com/AdoptOpenJDK/openjdk-jdk8u/blob/9ed418ee87883bcd73a144219a267a8d18277396/hotspot/src/share/vm/runtime/thread.cpp)），所以子线程执行完后会唤醒 Main 线程。
```java
    // Thread类join()方法的源码
    public final void join() throws InterruptedException {
        join(0);
    }

    // 同步方法，t1.join()表示获得了t1对象的锁，
    public final synchronized void join(long millis) throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                // 参数为0表示进入WAITING状态，直到其他线程唤醒
                // 调用者为线程对象a，a线程执行完后会唤醒等待在a上的其他线程
                wait(0);    
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                // join参数不为0，进入TIMED-WAITING状态表示等待一段时间或其他线程唤醒
                wait(delay);    
                now = System.currentTimeMillis() - base;
            }
        }
    }
```
```cpp
// Thread.cpp源码，可知线程执行完毕会唤醒其他线程
static void ensure_join(JavaThread* thread) {

  java_lang_Thread::set_thread_status(threadObj(), java_lang_Thread::TERMINATED);
  
  // to complete once we've done the notify_all below
  java_lang_Thread::set_thread(threadObj(), NULL);
  lock.notify_all(thread);      // 线程执行完毕，唤醒等待在thread对象上的其他线程
  thread->clear_pending_exception();
}
```
通过上面的源码，我们可以知道，join 底层原理就是 wait()，所以我们可以自己实现以下join方法
```java
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                // 子线程执行完毕，会自动调用notifyAll唤醒其他线程，源码Thread.cpp中
                System.out.println("子线程执行完毕，唤醒主线程");
            }
        });

        t.start();
        System.out.println("等待子线程运行完毕");
        
        // 下面三行代码与 t.join() 等价，获取线程t的Monitor锁，调用wait方法
        synchronized (t) {
            t.wait();
        }
        System.out.println("所有子线程执行完毕，开始执行Main线程");
    }
```
CountDownLatch 与 CylicBarrier
----
作用于join 类似且更加强大，参考《java并发编程的艺术》


> **面试题 14**：在 join 期间，线程会处于那种状态？（wait/notify，CountDownLatch都可以引到这个问题上来）
1. `join`期间，主线程处于`WAITING`状态，可以通过 debug 或 `mainThread.getState` 验证
2. 因为`join`底层是调用 `wait()` 方法，所以会处于`WAITING`状态
3. `wait()`方法获取的Monitor锁是线程对象的锁，子线程执行完会唤醒主线程

yield 方法
----

**作用：** 释放我的 CPU 时间片，线程状态依然是`RUNNABLE`。

一般开发中不会使用`yield`，但是JUC中AQS，ConcurrentHashMap，FutuerTask等都会使用到`yield`方法

sleep 会让出调度权，而`yield`虽然让出了调度权，但也随时可能被调度

其他Thread 方法
---
| Thread.currentThread()                            | 获取正在执行的线程对象     |
| ------------------------------------------------- | -------------------------- |
| getState()                                        | 获取线程状态               |
| getName()                                         | 获取线程名称               |
| interrupt()                                       | 发送中断通知               |
| isInterrupted()                                   | 获取中断标志位             |
| public Thread(Runnable target, String threadName) | 构造方法，可以设置线程名称 |

# 6. 线程的属性

| 线程属性        | 说明                                               |
| --------------- | -------------------------------------------------- |
| ID              | 每个线程都有自己的ID，用于标识不同的线程，不允许被修改           |
| 名称 Name       | 在开发过程中更容易区分不同线程，方便调试、定位问题 |
| 守护线程 isDaemon     | true表示是守护线程，false表示是用户线程            |
| 优先级 Priority | 告诉线程调度器，用户哪个线程多运行，哪个少运行     |




线程ID
---
查看 Thread 源码可知，线程ID从1开始，不允许被修改。

第一个是`Main`线程，因为 Main 是入口方法，同时还要启动若干个线程，如`Finalizer`线程用来执行对象的`finalize()`方法，`Reference Handler`线程用于处理GC相关
```java
    // 线程ID
    private static long threadSeqNumber;

    // 生成线程ID
    private static synchronized long nextThreadID() {
        // 先++后return，所以第一个线程Main线程ID为1
        return ++threadSeqNumber;
    }
```
线程名称
---
查看 Thread 源码可知，若未指定线程名称，则默**认使用 Thread+数字 作为线程名称**
```java
    // 线程的构造方法，若未指定线程名称，则默认使用 Thread+X 作为线程名称
    public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }
```

守护线程
---
**作用：** 为用户线程提供服务的线程称为守护线程，JVM 中没有了非`Daemon`线程，JVM需要退出，JVM中的所有`Deamon`线程都需要立即终止

**特性：** 
1. 线程类型默认继承自父线程
2. 除了Main线程，被JVM启动都是守护线程，用户启动的当然都是用户线程
3. 守护线程不影响JVM的退出，用户线程执行完后，JVM就会退出。

线程优先级
---
线程优先级共有 10 个级别，默认为 5。但是程序设计不应该依赖于优先级，因为不同的操作系统对优先级的处理不一样，比如 Windows 中线程只有7个优先级，Linux 中会忽略线程优先级；

> 面试题 15：如何利用线程优先级帮助程序运行，有哪些禁忌？

因为不同操作系统对优先级的处理不同，不一定能生效，比如Win有7个级别，而Linux会忽略线程优先级

# 7 线程的异常

主线程可以轻松发现异常，而子线程发生异常却很难发现

子线程异常无法用传统方法捕获

> 如何全局处理异常，为什么要全局处理，不处理可以吗？
> run 方法是否可以抛出异常？如果抛出异常，线程状态会怎么样？
> 线程中如何处理某个未处理异常？


# 8 线程安全

> 什么是线程安全？
> 
当多个线程访问一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那这个对象就是线程安全的 ---- Brin Goetz


# 9 Java 内存模型JMM


彩蛋：自顶向下的好处
---
先讲适用场景，再讲怎么用，最后讲原理。问题兴趣驱动，与传统的教育方式相反

直观的理解，感性的认识，有助于加深理解，最后带着好奇心去分析源码
《计算机网络 自顶向下方法》

> 为什么需要JMM（Java Memory Model）？

因为不同的 CPU 平台的机器指令千差万别，无法保证Java 代码到 CPU 指令翻译的准确无误，所以需要JMM来统一规范，让多线程运行的结果可预期

**并发编程有三个问题：**
1. 缓存导致的可见性问题 - happen-before规则，volatile也能禁用CPU本地缓存
2. 编译优化带来的重排序问题 - volatile语义增强禁止重排序
3. 线程切换带来的原子性问题 - 互斥锁来禁止线程切换

所以为了解决上述问题，JMM 分为三个部分：**重排序、可见性、原子性**

**本章节详细内容参考《Java并发编程的艺术》第3章**


## 9.1 重排序
重排序带来的问题
---
先来看一段代码，由于 CPU 执行多线程时不断切换，有可能得到 4 中结果
1. x=1，y=0   t2先执行，t1再执行
2. x=0, y=1   t1先执行，t2再执行
3. x=1, y=1   t1给a赋值完后 CPU 切换到 t2执行
4. x=0, y=0   由于内存重排序，t1对a进行了赋值`a=1`，但没有将该数据刷新到**主存**，导致t2执行`y=a`时从主存拿到的`a==0`，所以最终y=0；同理，t2 执行的`b=1`也没有刷新到主存，这就是**重排序**带来的问题。
```java
public class OutOfOrderDemo {
    private static int a = 0, b = 0;
    private static int x = 0, y = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
                    a = 1;
                    x = b;
        });
        Thread t2 = new Thread(() -> {
            b = 1;
            y = a;
        });

        // 启动线程，交换线程的启动顺序，会得到不同的x,y
        t1.start();
        t2.start();
        // 等待两个线程执行完毕
        t1.join();
        t2.join();
        System.out.println("x = " + x + ", y = " + y);
    }
}
```
![20200110054241.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200110054241.png)

![20200110060158.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200110060158.png)
上图处理器A和处理器B可以同时把共享变量`a=1,b=1`写入自己的缓冲区（A1，B1），然后从主存中读取另一个共享变量`b=0,a=0`(A2,B2),最后才把自己写缓冲区保存的脏数据`x=0,y=0`刷新到了主存中（A3,B3），虽然处理器A的执行顺序是A1->A2，但内存操作实际发生的顺序是A2->A1，此时，处理器A的内存操作就被重排序了。（详见Java并发编程的艺术P25）


为了演示`x=0, y=0`的情况，可以[查看示例代码](https://github.com/maoturing/java_concurrency_core/blob/master/src/jmm/OutOfOrderDemo2.java)，需要多次运行，运行结果如下图所示

![20200110140511.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200110140511.png)

**重排序分为三种情况：**
1. 编译器优化：包括JVM，JIT编译器
2. CPU 指令重排：
3. 内存重排序

![并发编程的艺术P24.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200110054334.png)

重排序的好处
---
![20200110051036.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200110051036.png)
如上图所示，CPU 对代码进行了重排序，使得原先 9 条指令优化为了 7 条指令，**提高了 CPU 的处理速度**。其中 Load 表示从内存读取到CPU，Set 表示赋值，Store 表示存储到内存

## 9.2 可见性
因为CPU 有多级缓存，导致读的数据会过期。

高速缓存的容量比主内存小，但是速度仅次于寄存器，所以CPU和主内存之间就多了Cache层

线程间对于共享变量的可见性问题不是直接由多核引起的，而是多级缓存引起的
![20200110122111.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200110122111.png)

由上图可知，CPU 有多级缓存，JMM将共享的内存`L3 Cache`、`RAM`抽象为**主存**，将核心独有的内存`registers`（寄存器）、`L1 chache`、`L2 cache`抽象为**本地内存（工作内存）**

> 为什么需要多级缓存？

为了提高CPU的执行速度，因为CPU的速度远远高于内存，所以需要将内存中的数据提前读取到缓存中。

比如我们找一本书的过程，书桌上有常用的书，数量少，找书速度快，如果书桌上找不到，那么我们就去校图书馆找，数量较大速度一般，如果还找不到，就去市图书馆去找，数量极大速度最慢。书桌就是一级缓存，校图书馆就是二级缓存，市图书馆是内存，这三者容量依次升高，查找速度依次降低。虽然一级缓存速度最高，但也不能指望提高一级缓存的大小来提高缓存读取速度，因为缓存越大，查找速度越慢。

另外将内存中的数据读到内存中，但是CPU需要的数据不一定在缓存中，这里涉及**缓存命中**（此处添加缓存命中文章的链接）的知识



主存与本地内存的关系
---
JMM 有以下规定：
1. 所有变量都存储在主存中，同时每个线程也有自己的本地内存，工作内存中的变量内容是主存中的拷贝
2. 线程不能直接读写主存中的变量，只能操作本地内存中的变量，然后再同步到主存中
3. 主存是多个线程共享的，但线程间不共享本地内存，如果线程间需要通信，必须借助主存中专来完成

Happen-Before （先行发生）原则
---
> **如果说操作A Happen-Before 于操作B，其实就是说操作B之前，操作A的影响对于操作B可见**
> 
Happen-before 的概念来阐述操作之间的内存**可见性**，`happen-before`仅要求第一个操作执行结果对第二个操作可见，且前一个操作实际执行时间排在第二个操作之前（the first is **visible** to and ordered before the second）

在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在`happen-before`关系，详细见《并发编程艺术》p26，《深入浅出JVM》p376，一共有 9 个规则，其中**最重要的是前 4 条规则**

1. **单线程原则：一个线程**中的每个操作，对于该线程中的任意后续动作可见，也就是说**都在本地内存运行，不存在可见性问题**，又叫程序顺序原则
2. **`Monitor`锁原则**（synchronized和Lock）：对一个锁的解锁，对于后续其他线程**同一个锁**的加锁可见，这里的“后续”指的是时间上的先后顺序，又叫管程锁定原则，
3. **`volatile` 变量原则**：对于一个volatile变量的写，对于后续其他线程对volatile变量的读可见，也就是说**volatile变量的写会直接刷新到主存**，这里的“后续”指的是时间上的先后顺序
4. **传递性原则**：如果A happen-before B，B happen-before C，那么A happen-before C，最常用到的一个原则
5. **`start()`原则**：如果线程A执行ThreadB.start()启动线程B，那么A线程ThreadB.start()及之前的操作对于线程B的所有操作可见
6. **线程终止原则**：线程中的所有操作，对于此线程的终止检测可见，我们可以通过`Thread.join()`方法结束，`Thread.isAlive()`的返回值等手段，检测到线程终止运行
7. **`join()`原则**：如果线程A执行操作ThreadB.join()**并成功返回**，那么线程B中的所有操作，对于线程A可见，实际开发中，我们也是用`join()`方法来获取线程B中的执行结果，这一条规则其实是线程终止原则的细化部分
8. **线程中断规则**：对线程A 调用`ThreadB.interrupt()`方法，对于线程B检测中断`isInterrupted`可见
9.  **对象终结原则**：一个对象的初始化完成，对于该对象的`finalize()`方法可见

![20200110125749.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200110125749.png)

上图中由**程序顺序原则**可知，`1 happen-before 2`，`3 happen-before 4`，由**`volatile` 变量原则** 可知，`2 happen-before 3`，再结合**传递性原则**，可知 `1 happen-before 4`
另外还有一个重要的并发工具类原则：

1. 线程安全的容器，如`CurrenthashMap`的`put`操作，对于后续`get`操作可见
2. `CountDownLatch`的`countDown()`操作，对于后续`await()`可见
3. `Semaphore`的`release()`释放许可证操作，对于后续`acquire()`获取许可证操作可见
4. `CyclicBarrier`的最后一个线程到达屏障时，对于所有被拦截的线程`await()`可见
5. `Future`的`call()`操作执行结果，对于后续`get()`操作可见，详细见[示例](https://github.com/maoturing/java_concurrency_core/blob/master/src/threadcoreknowledge/createthreads/wrongways/CallableStyle.java)
6. 线程池

>面试题： 对于一个锁的unlock操作，对于后续的lock操作可见。因为如果不可见，其他线程就没法获取锁了。那对于一个锁的lock操作，对于后续的unlock肯定也是可见的，那么这个由什么保证呢？

对于一个锁的unlock操作，对于后续的lock操作可见，隐含条件是对其他线程后续的lock操作可见。而对于一个锁的lock操作和unlock操作，肯定是在同一个线程里，可以用**单线程原则**解释
> 面试题：volatile 变量的写 happen-before volatile 变量的读是怎么保证的？
> 
插入内存屏障，每个volatile写操作的前面都会插入`StoreStore`屏障，后面都会插入一个StoreLoad屏障，如下图所示

![20200111080248.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200111080248.png)

`StoreStore`屏障保证在volatile写之前的普通写操作，已经对任意处理器可见了。因为`StoreStore`屏障把上面的所有普通写刷新到了内存
详细见（详细见《Java并发编程的艺术》3.4.4）





## volatile 关键字

`volatile`是一种同步机制，比synchronized或者Lock相关类更轻量，因为使用volatile不会发生**上下文切换**等开销很大的行为。

需要注意volatile做不到synchronized那样的原子保护

**不适应场景：i++**

不是一个原子操作，volatile无法保证原子性，导致出错

**使用场景1：boolean flag**

如果一个共享变量自始至终只被各个线程赋值，没有其他操作，则可以使用volatile来替代synchronized和原子变量，因为赋值本身就是原子操作，而volatile又保证了可见性，所以线程安全

**使用场景2：作为刷新之前变量的触发器**

volatile能保证之前的操作全部刷新到主存 

下面的代码，变量 v 的作用就是触发器，当v=true时，前面的代码（x=42)的执行结果已经刷新到了主存

> 面试题：线程A执行完`writer()`后，线程B执行`reader()`，下面代码注释部分x会是多少呢？为什么

```java
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void writer() {
    x = 42;                 // 1
    v = true;               // 2
  }
  public void reader() {
    if (v == true) {        // 3
      // 这里x会是多少呢？
      int i = x;            // 4
    }
  }
}
```
根据`happen-before`单线程原则，1 `happen-before` 2，3 `happen-before` 4；volatile变量原则，2 `happen-before` 3；再结合传递性原则，1 `happen-before` 4，所以得到i=42

![20200111071300.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/20200111071300.png)

在旧的内存模型中，当1和2之间没有数据依赖关系时，1和2之间就可能被重排序。其结果就是线程B执行到4时，不一定能看到线程A对共享变量x的修改，x可能为0。
JSR133之前对下面代码重排序不进行限制，JSR133增强了volatile的内存语义，严格限制编译器和处理器对volatile变量与普通变量的重排序，所以x=42。（详细见《Java并发编程的艺术》3.4.5）

x=45; // 1
v=true; // 2

> 

volatile的两点作用
-----

**可见性：**读一个volatile变量之前，需要先使相应的本地缓存失效，这样就必须到主内存读取最新值，写一个volatile属性会立即刷入到主内存

**禁止指令重排序：**解决单例双重锁乱序问题

 

volatile小结
----

1. volatile属性的读写操作都是无锁的，它不能替代synchronized，因为它没有提供原子性和互斥性。因为无锁，不需要花费时间在获取锁和释放锁，所以说volatile是低成本的

2. volatile只能作用于属性，我们用volatile修饰属性，这样能禁止重排序

3. volatile提供了可见性，任何一个线程对其的修改对其他线程立即可见。volatile属性不会使用本地缓存，始终从主存中读取和写入
4.  volatile 提供了Happen-before保证，对volatile变量v的写入happen-before所有其他线程后续对v的读取
5. volatile 可以使得 long 和double 变量的赋值是原子操作

保证可见性的几种方法
------

volatile、synchronized、Lock、并发集合、Thread.join()、Thread.start()

Happen-before


> 面试题：有一个共享变量 abc，在一个线程里设置了 abc 的值 abc=3，你思考一下，有哪些办法可以让其他线程能够看到abc==3？

见极客时间



synchronized
----

synchronized不仅保证了原子性，还保证了可见性



## 9.3 原子性

一系列的操作，要么全部执行成功，要么全部不执行，不会出现执行一般的情况

Java中的原子操作
---

- 基本类型（int，byte，boolean，short，char，float）的赋值操作，除了long和double，i++不是原子操作

- 所有引用的赋值操作，不管是32位还是64位机器

- java.concurrent.Atomic.*包中所有类的原子操作

long和double的原子性
----
1. **有可能出现线程安全问题**，32位机器上对64位的long/double类型变量进行读写操作，可能出现线程安全问题。Java语言规范鼓励但不强求JVM对64位的long和double类型变量的写操作具有原子性，所以存在不是原子操作的可能

1. **不需要加volatile**，JSR对于商用的JVM，强烈建议将load, store, read, write四个操作实现为原子操作，而且目前各平台下的**商用JVM都将其实现为了原子操作**。因此实际编程中不需要把long，double类型修饰为volatile变量。

详细见《深入JVM》



全同步的HashMap的线程安全问题，待补充

原子操作的实现原理
-----
见《Java并发编程艺术》2.3原子操作的实现原理。
处理器使用一下两种方法实现原子操作
1.使用总线LOCK#信号保证原子性
2.通过缓存锁定保证原子性

Java 使用CAS实现原子操作
Atomic的原理就是CAS，CAS操作会带来三个问题：1. ABA 2. 循环时间长开销大  3. 只能保证一个共享变量的原子操作

>  面试题：volatile修饰的变量 i，能保证`i++`操作线程安全吗?

经过[示例代码]()github.jmm.NOVolaitile验证，不能保证线程安全，因为`i++`不是原子操作，需要4步才能完成，而volatile仅能解决重排序和可见性问题，原子性问题需要锁或CAS（原子类）来解决，详细见《码书》p230

## 9.4 单例模式

为什么需要单例模式?

1. **节省内存CPU资源**，如果创建一个需要耗费大量内存、大量计算（耗费CPU资源），大量耗时（从DB读取数据）的对象，我们使用单例模式，可以节省内存CPU资源
2. **保证结果正确**，如多线程统计访问人数，需要一个**全局的计数器**实例，如果创建了多个计数器，就会统计错误，需要代码层面**限制创建多个计数器对象实例**
3. 方便管理，如日期工具类、字符串工具类，我们不需要创建多个工具类对象实例，只会耗费内存，一般工具类都是`类.静态方法`调用，也需要代码层面**限制创建工具类实例**，如`java.lang.Math`类的构造方法都是私有的，`java.lang.Runtime`也是单例模式

**单例模式适用场景**

1. **无状态的工具类：**如日志工具类，不管在哪里适用，只需要它记录日志信息，并不需要它的实例对象上存储任何状态，故我们只需要一个实例对象即可。spring无状态bean，
2. **全局信息类：**比如一个类用来统计网站的访问次数，我们不希望有的访问记录在对象A上，有的记录在对象B上，我们就让这个类成为单例



**spirng框架使用单例模式**

对于最常用的spring框架来说，我们经常用spring来帮我们管理一些无状态的bean，其默认设置为单例，这样在整个spring框架的运行过程中，即使被多个线程访问和调用，这些“无状态”的bean就只会存在一个，为他们服务。那么“无状态”bean指的是什么呢？

**无状态：**当前我们托管给spring框架管理的javabean主要有service、mybatis的mapper、一些utils，这些bean中一般都是与当前线程会话状态无关的，没有自己的属性，只是在方法中会处理相应的逻辑，每个线程调用的都是自己的方法，在自己的方法栈中。

**有状态：**指的是每个用户有自己特有的一个实例，在用户的生存期内，bean保持了用户的信息，即“有状态”；一旦用户灭亡（调用结束或实例结束），bean的生命期也告结束。即每个用户最初都会得到一个初始的bean，因此在将一些bean如User这些托管给spring管理时，需要设置为prototype多例，因为比如user，每个线程会话进来时操作的user对象都不同，因此需要设置为多例。

**优势：**

1. 减少了新生成实例的消耗，spring会通过反射或者cglib来生成bean实例这都是耗性能的操作，其次给对象分配内存也会涉及复杂算法；
2. 减少jvm垃圾回收；
3. 可以快速获取到bean；

**劣势：**

单例的bean一个最大的劣势就是要时刻注意线程安全的问题，因为一旦有线程间共享数据变很可能引发问题。

**log4j中的单例模式**

在使用log4j框架时也注意到了其使用的是单例，当然也为了保证单个线程对日志文件的读写时不出问题。如果是多例，那么后面的实例日志操作会覆盖之前的日志文件



参考慕课网《Java设计模式》单例模式的实践完善本小节

单例模式的8种实现
----

1. 饿汉式
2. 懒汉式
3. 双重检查
4. 静态内部类
5. 枚举

双重检查单例模式
----

双重检查单例模式的实现如下所示，`INSTANCE = new DoubleCheckSingleton();`不是原子操作，分为三个步骤：

1. 分配堆内存
2. 对象初始化，调用构造方法创建对象
3. 将对象赋值给INSTANCE变量

其中步骤2和步骤三可能出现重排序，如果执行顺序为132，当执行到步骤2时，另一个线程进入，发现INSTANCE不为NULL，则会返回一个未初始化完毕的实例对象

![1.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/1.png)
```java
/**
 * 双重检查单例模式, 推荐使用
 * 线程安全, 延迟加载, 效率高
 */
public class DoubleCheckSingleton {

    private volatile static DoubleCheckSingleton INSTANCE;
    /**
     * 构造函数私有, 避免破坏单例
     */
    private DoubleCheckSingleton(){};

    /**
     * 获取单例对象, 需要两次if检查, 故称为双重检查
     * 解决了LazyUnSyncSingleton线程不安全的问题, 解决了LazySyncSingleton后续获取对象的效率低的问题
     *
     * 但是 new DoubleCheckSingleton() 不是一个原子操作, 当另一个线程进入第一次检查if(null == INSTANCE), 会返回一个未初始化完成的实例对象
     * 所以需要volatile 来禁止重排序
     */
    public  static DoubleCheckSingleton getInstance() {
        if(null == INSTANCE) {
            synchronized (DoubleCheckSingleton.class) {
                if(null == INSTANCE) {
                    // 不是原子操作,需要volatile禁止重排序
                    INSTANCE = new  DoubleCheckSingleton();
                }
            }
        }
        return INSTANCE;
    }
}
```
静态内部类单例模式
----
实现原理见JVM书，懒加载，用JVM类加载特性保证线程安全

<https://blog.csdn.net/mnb65482/article/details/80458571>
单例模式静态内部类原理




枚举单例模式
----

《Effective Java》说使用单元素的枚举是实现单例模式的最佳方法 

1. 写法简单
2. 线程安全
3. 避免反序列化破坏单例
4. 避免反射攻击



验证反射是否能够破坏枚举模式，示例代码如下，会报NoSuchMethodException异常，详细原理见参考文档12

```java
    EnumSingleton singleton1=EnumSingleton.INSTANCE;
    EnumSingleton singleton2=EnumSingleton.INSTANCE;
    System.out.println("正常情况下，实例化两个实例是否相同："+(singleton1==singleton2));
    Constructor<EnumSingleton> constructor= null;
    constructor = EnumSingleton.class.getDeclaredConstructor();
    constructor.setAccessible(true);
    EnumSingleton singleton3= null;
    singleton3 = constructor.newInstance();
   	     System.out.println(singleton1+"\n"+singleton2+"\n"+singleton3);
    System.out.println("通过反射攻击单例模式情况下，实例化两个实例是否相同："+(singleton1==singleton3));
```






>  面试题：双重检查单例模式的特点

优点：线程安全，延迟加载，获取对象效率高

1. 为什么要double-check？
   - synchronized修饰方法线程安全但后续获取实例效率低
   - synchronized缩小范围，单check线程不安全

为了兼顾线程安全和后续获取实例的效率，衍生出来双重检测单例模式

2. 为什么要用volatile？

   - 新建对象不是原子操作，需要分类内存，调用构造方法，赋值操作三部分

   - 重排序可能会使得赋值操作早于调用构造方法，出现NPE，所以需要volatile禁止重排序

实践
​	tsp中使用单例模式，没有使用volatile，且创建对象后还要调用set方法，不是原子操作，会出现线程安全问题，解决办法见[印象笔记](<https://app.yinxiang.com/shard/s64/nl/21168998/81d9f8a3-066d-4004-b54e-16d00ebb14c2/>)



> 面试题：单例模式的最佳实现是什么？

 1. Effective Java中说枚举是单例的最佳实现
 2. 写法简单
 3. 线程安全
 4. 避免反序列化破坏单例

>  面试题: 单例模式实现有几种，各有哪些优缺点？

  静态内部类的实现方式可以引申到JVM类加载

  双重检查的实现方式可以引申到并发、锁、volatile、重排序、原子操作等知识

  枚举类的实现方式可以引申到反编译，枚举的原理，反序列化
> 面试题：什么是JMM，JMM为了解决什么问题？

> 面试题：Java内存模型
Happen-before volatile 主存和本地缓存

> 面试题：volatile和synchronized的异同

> 面试题： 什么是原子操作，i++、创建对象、赋值、long类型的写是不是原子操作，怎么解决？

> 面试题：什么是内存可见性？
![内存可见性211.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/内存可见性211.png)

> 面试题：64位的double和long写入操作是原子操作吗？

1. **有可能**，Java语言规范鼓励但不强求JVM对64位的long和double类型变量的写操作具有原子性，所以存在不是原子操作的可能
2. **不需要加volatile**，JSR对于商用的JVM，强烈建议将load, store, read, write四个操作实现为原子操作，而且目前各平台下的**商用JVM都将其实现为了原子操作**。因此不需要把long，double类型修饰为volatile变量。

# 10 死锁

考考你

1. 写一个必然死锁的例子(百度面试题)
2. 发生死锁必须满足哪些条件?
3. 如何定位死锁?
4. 有哪些解决死锁问题的策略
5. 讲讲经典的哲学家就餐问题
6. 实际工程中如何避免死锁?
7. 什么是活跃性问题? 活锁、饥饿和死锁有什么区别？

>  死锁是什么？

当两个（或更多）线程（或进程）相互持有对方所需要的资源，又不主动释放，导致所有线程都无法继续运行，陷入无尽的阻塞，这就是死锁

## 10.1 死锁的影响

数据库中：两个事务互相持有对方需要的资源，检测到死锁后会放弃事务，然后指派一个事务先放弃，释放资源，另一个事务执行后再执行该事务

JVM中：无法自动处理，因为不确定线程的重要性，所以JVM无法指派一个线程先放弃。但是JVM提供检测死锁的功能，将处理权利交给程序员

死锁代码
```java

```

上述代码运行会进入死锁，一直等待无法结束，手动停止线程后会打印如下信息
```shell
Process finished with exit code 130(interrupted by signal 2:SIGINT)
```

正常退出exit code为0，但是不确定130是否是死锁退出的标志

**死锁产生的四大条件**

1. 互斥条件：共享资源X和Y只能被一个线程占有
2. 占有且等待条件：线程T1已经获得共享资源X，在等待共享资源Y时，不释放X
3. 不剥夺条件：不能剥夺线程T1已经获得的共享资源
4. 循环等待条件：线程T1等待T2占有的资源；线程T2等待T1占有的资源



## 10.2 定位死锁

`jstack [pid]`：通过jps获得进程pid，然后使用jstack命令，获取死锁信息如下

![死锁信息.png](https://raw.githubusercontent.com/maoturing/PictureBed/master/pic/死锁信息.png)


ThreadMXBean：通过代码检测死锁，只能检测当前进程中的死锁
```java
ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
        if (deadlockedThreads != null && deadlockedThreads.length > 0) {
            for (int i = 0; i < deadlockedThreads.length; i++) {
                ThreadInfo threadInfo = threadMXBean.getThreadInfo(deadlockedThreads[i]);
                System.out.println("发现死锁" + threadInfo.getThreadName());
            }
        }
```

## 10.3 修复死锁

修复死锁的思路就是**破坏死锁产生的四大条件**，常用的解决方案有3种：

1. 避免策略：哲学家就餐的换手方案，转账换序方案
2. 检测与恢复策略：定时检测是否存在死锁，如果有就**剥夺**某一个资源，来打开死锁
3. 鸵鸟策略：如果死锁发生概率极低，可以直接忽略它，直到死锁发生的时候再人工修复

**避免策略**
----
思路：避免相反的获取锁的顺序。如转账时需要获取转出账户、转入账户两把锁，但是实际上不在乎获取锁的顺序，当两把锁都获取到了才能进行转账操作。所以可以避免两个线程产生死锁。

通过hashcode来决定获取锁的顺序，hashcode相同时需要“加时赛”，如果对象锁有主键，则利用主键替代hashcode更方便

下面转账代码中，两个线程互相转账，操作前要获取两个Account对象锁，分别是from和to，**哪个锁的hash值小，哪个锁先被获取**，这样两个线程都先请求获取hash值小的锁，获取到了才能获取另一把锁，这样就不会产生死锁了。破坏了死锁产生条件中**循环等待条件**。即使是更加复杂的多人随机转账产生，因为**无法形成闭环**了，也不会产生死锁。

```java

```



哲学家就餐问题
----

哲学家就餐问题本质是一个**死锁问题**，解决哲学家就餐问题，就是解决死锁问题。



问题描述：五个哲学家在一张桌子上吃饭，两人之间有一只筷子，共5只筷子，哲学家就餐流程：

1. 先拿起左手边的一只筷子

2. 然后拿起右手边的一只筷子

3. 如果筷子正在被别人使用，那就等待别人用完

4. 拿到两只筷子后开始吃饭，吃完后将筷子放回

**死锁：**每个哲学家都拿着左手的筷子，永远都在等待右边的筷子，就会陷入一直等待的状态

解决办法：

1. **服务员检查（避免策略）**：每个哲学家**拿左手边筷子前**先询问服务员，当服务员发现其他4个哲学家都有且仅有左手边筷子时，即这个哲学家拿起左手筷子就会死锁，为了防止死锁，服务员不允许这个哲学家拿左手边筷子。
2. **改变一个哲学家拿筷子的顺序（避免策略）**：因为都拿到了左手边筷子，都在请求右手边筷子，形成了一个闭环，只需要改变其中一个哲学家拿筷子的顺序，即可打破这个闭环
3. **餐票（避免策略）**：因为5个人同时拿到左手筷子会发生死锁，所以只提供4张餐票，即最多有4个人拿筷子就餐，即可避免死锁问题
4. **领导调节（检测与恢复策略）**：与避免策略不同，该策略不避免你发生死锁，当发生死锁后，检测出死锁，领导命令其中一个人放下筷子，破坏了死锁形成四个条件中的不剥夺条件，解决了死锁问题

**死锁检测算法**：每次获取锁都有记录，检查锁的调用链路图，如果存在环路，则说形成了死锁

```java
/**
 * 描述：     演示哲学家就餐问题导致的死锁
 * 解决办法: 改变一个哲学家拿筷子的顺序
 */
public class DiningPhilosophers {
	// 哲学家类
    public static class Philosopher implements Runnable {

        private Object leftChopstick;

        public Philosopher(Object leftChopstick, Object rightChopstick) {
            this.leftChopstick = leftChopstick;
            this.rightChopstick = rightChopstick;
        }

        private Object rightChopstick;

        @Override
        public void run() {
            try {
                // 先拿左边筷子,再拿右边筷子,吃完后放下筷子
                while (true) {
                    doAction("Thinking");
                    synchronized (leftChopstick) {
                        doAction("Picked up left chopstick");
                        synchronized (rightChopstick) {
                            doAction("Picked up right chopstick - eating");
                            doAction("Put down right chopstick");
                        }
                        doAction("Put down left chopstick");
                    }
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        private void doAction(String action) throws InterruptedException {
            System.out.println(Thread.currentThread().getName() + " " + action);
            Thread.sleep((long) (Math.random() * 10));
        }
    }

    public static void main(String[] args) {
        // 总共5个哲学家,5只筷子
        Philosopher[] philosophers = new Philosopher[5];
        Object[] chopsticks = new Object[philosophers.length];
        for (int i = 0; i < chopsticks.length; i++) {
            chopsticks[i] = new Object();
        }
        for (int i = 0; i < philosophers.length; i++) {
            Object leftChopstick = chopsticks[i];
            Object rightChopstick = chopsticks[(i + 1) % chopsticks.length];
            // 如果是最后一位哲学家, 则先拿右手边筷子, 避免了环路的形成
            if (i == philosophers.length - 1) {
                philosophers[i] = new Philosopher(rightChopstick, leftChopstick);
            } else {
                philosophers[i] = new Philosopher(leftChopstick, rightChopstick);
            }
            new Thread(philosophers[i], "哲学家" + (i + 1) + "号").start();
        }
    }
}

```




​	


检测与恢复策略
---

检测：每次获取锁都有记录，定期检查锁的调用链路图，如果存在环路，则说形成了死锁，一旦出现死锁，就用死锁恢复机制进行恢复。

恢复方法有两种：	

线程终止，逐个终止线程，直至死锁消除

资源抢占，把已经分发的锁给收回来，让线程回退几步，这样就不用结束整个线程

??? 两种恢复方法实际如何操作并不清楚

## 10.4 如何避免死锁

1. 设置超时时间

Lock.tryLock(long timeout, TimeUnit unit)  尝试获取锁，获取成功返回true，超时后放弃返回false

示例代码如下：	

```java


```



synchronized不具备尝试锁的能力

获取锁失败：打印日志，发送报警信息、重启等

2. 多使用并发类而不是自己设计锁
3. 尽量降低锁的粒度
4. 同步代码块优于同步方法，自己制定锁对象更好
5. 线程设置一个有意义名称，debug和排查问题事半功倍，框架和JDK都遵守这个规则
6. 尽量避免锁的嵌套
7. 分配资源前先看能不能收回来：**银行家算法**
8. 尽量不要多个功能用同一把锁：专锁专用

## 10.5 活锁

**会导致程序无法顺利进行，统称为活跃性问题**。**死锁**是最常见的活跃性问题，**活锁**（LiveLock）和**饥饿**都是活跃性问题。

**什么是活锁？**

虽然线程并没有阻塞，也始终在运行，但是程序却得不到进展，因为线程始终在做同样的事。

活锁对应到哲学家就餐问题：

五个哲学家都拿到了左边的筷子，都在等待右边的的筷子，最多等待5分钟，如果拿不到右边的筷子，就放下手中的筷子，再等五分钟，又同时拿起左手边的筷子



工程中的活锁实例

消息队列中的消息如果处理失败，不能放在队列开头重试，应该放到队列尾部，设置重试次数，如果还是失败，可以考虑保存到数据库或写到文件中

## 10.6 饥饿

当线程需要某些资源（例如CPU），但是始终得不到，称为**饥饿**。



>  面试题：写一个必然死锁的例子，生产中什么场景会产生死锁？

线程a获得锁1，请求锁2,；线程b获得锁2，请求锁1

>  面试题：发生死锁必须满足哪些条件？

四大条件：互斥条件、占有且等待条件、不剥夺条件、循环等待条件

> 面试题：如何定位死锁？发现死锁的原理是什么？

jstack命令、jconsole、ThreadMXBean都可以定位死锁

发现死锁的原理是根据锁的调用链图，形成闭环则说明形成了死锁

> 面试题：有哪些解决死锁的策略？

避免策略：哲学家就餐的换手方案，转账换序方案，根据账户ID确定获取锁的顺序

检测与恢复策略：一段时间检测是否有死锁，如果有就剥夺某一个资源，来打开死锁

> 面试题：哲学家就餐问题

哲学家就餐死锁问题有四种解决办法：服务员检查（避免策略）、改变一个哲学家拿筷子的顺序（避免策略）、餐票（避免策略）、领导调节（检测与恢复策略）

> 面试题：实际工程中如何避免死锁？

1. 设置等待锁的超市时间 Lock.tryLock()

2. 多使用并发类而不是自己设计锁

   ...共8点，见上方详解

> 面试题：什么是活跃性问题? 活锁、饥饿和死锁有什么区别？



# 11 总结

使用锚点整体面试题

<a name="q1">面试题1</a>

<a href="#q1">跳转到面试题1</a>





# 12 面试题归纳与套路


单例，分析双重锁，引出线程安全和和volatile禁止重排序，结合TSP实践说明单例模式。再引出Spring单例，再引出cglib，再引出ThreadLocal，结合注解记录日志说明ThreadLocal

分析静态内部类，引出类加载方式

分析原子操作，引出i++不是线程安全，引出Atomic，引出CAS

分析原子操作，引出锁，解释sync'原理，引出死锁，结合WG死锁检测实践，再引出死锁调用链形成闭环检测死锁，再引出哲学家就餐问题

> 都有哪些方法会抛出InterruptedException?

# 参考文档

1. [Java并发核心知识体系精讲 - 视频教程](https://coding.imooc.com/class/362.html)
2. [线程8大核心基础 - 思维导图](http://naotu.baidu.com/file/07f437ff6bc3fa7939e171b00f133e17?token=6744a1c6ca6860a0)
3. [配套高频并发面试题汇总 - 持续更新]()
4. [OpenJDK 在线代码](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/b860bcc84d51/src/share/)：class 目录是 Java 代码，native 目录是 C++ 代码
5. [OpenJDK 源码 - Github](https://github.com/AdoptOpenJDK/openjdk-jdk8u)
6. [Java线程源码解析之 start](https://www.jianshu.com/p/81a56497e073)
7. [Java线程面试题](https://www.cnblogs.com/bsjl/p/7693029.html?tdsourcetag=s_pctim_aiomsg)
8. [JVM源码分析之Object.wait/notify实现 - 占小狼](https://www.jianshu.com/p/f4454164c017)
9. [123个Java并发面试题](https://www.bilibili.com/read/cv4357944)
10. [Java 单例模式详解](https://blog.csdn.net/LS7011846/article/details/100068311)
11. [Java单例模式的5种写法](<https://mp.weixin.qq.com/s/dU_Mzz76h-qQZvrgeSe44g>)
12. [枚举实现单例模式的原理](https://www.cnblogs.com/saoyou/p/11087462.html)



结合[极客时间](https://time.geekbang.org/column/intro/159)、汪文君并发、Java并发编程的艺术，微信收藏、印象笔记、简书笔记等完善，不完善总结相当于白学了
学完后过一下参考文档9面试题，检验一下学习成果。

局部性原理笔记




# 错误总结
