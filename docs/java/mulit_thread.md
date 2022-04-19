# Mulit Thread(多线程)

## Java内存模型(JMM)

Java内存模型时共享内存的并发模型，线程之间主要通过读-写`共享变量`来完成通信。Java内存模型控制着Java线程间的通讯，决定一个线程对共享变量的写入何时对另一个线程可见。

![jmm_java内存模型](./images/jmm_java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.png)

### 主内存与工作内存

Java内存模型中规定了所有的变量都存储在主内存中，每条线程还有自己的工作内存，线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的变量。

工作内存的出现，目标是为了提升线程对变量内存操作的效率；但也是造成`可见性`问题的主要根源之一：在不基于一定保证可见性手段的情况下，我们是无法确定各个线程是何时将工作内存的共享变量刷新到主内存中的，导致一个指令的操作结果对另一个操作时不可见的。

### Happens-Before规则

如果我们在并发编程中，一个指令的操作结果需要对另一个操作是`一定`可见，那么这两个操作之间必须存在`Happens-Before`的关系。这两个操作既可以时同一个线程中的，也可以不是同一个线程中的。

存在`Happens-Before规则`的目的：我们不是所有的并发编程中都需要考虑到可见性的问题；但是当我们需要处理可见性问题时，需要有一个原则进行指导，使我们决定用何种方式实现可见性。

**JMM中Happens-Before规则对程序员的承诺：** 如果基于某种手段实现了`A Happens-Before B`，那么JMM向程序员保证，A指令的结果一定是对B是立刻可见的，且A指令的执行顺序一定是先于B指令的。

**JMM中Happens-Before规则对编译器的约束：** 只要不改变程序的执行结果，编译器和处理器怎么优化都行。但是，如果某两个操作有`Happens-Before`的关系，编译器就不能打破这个约束，对两个指令进行重排序。

### volatile变量

volatile关键字就是践行`Happens-Before规则`的实现。作用如下：

- 使用volatile关键字修饰的变量，其写操作一定happens-before于任意后续对这个volatile域的读。这样避免了共享变量的读写被编译器重排序。（一定是先完成写，后续再进行读；否则，先进行读，再写，还是会来回覆盖，永远读不到一个最新的值操作）

- 使用volatile关键字修饰的变量，会将写缓冲区的内容同步刷新到主内存，保证其他线程或CPU可见；其他线程或CPU读取的时候，也是直接从主内存读取

基于volatile关键字以上两点功能，我们才在并发编程中实现了共享变量的安全可见，缺一不可。如果只是单独把共享变量写入到主存中，但是对于变量读写的重排序没有禁止，仍然无法保证可见性。

**参考**
> Java内存模型（JMM）总结: https://zhuanlan.zhihu.com/p/29881777
>
> Java并发编程之happens-before: https://www.cnblogs.com/senlinyang/p/7875458.html
>
> 读书笔记：从happens-before原则说起: https://blog.csdn.net/zdxiq000/article/details/60874848
>
> Java内存区域（运行时数据区域）和内存模型（JMM）: https://mp.weixin.qq.com/s/szsvz7Sfn_6xerkrQyNj4w

----

## 线程(Thread)

### Java线程是用户态还是内核态

- Java线程理论上说仍然是用户态线程，不过它去使用内核线程的高级接口-轻量级进程(Light Weight Process，LWP)，实现在内核线程上的运行

- Java线程和内核线程采用1比1的映射模型：即1个Java线程映射着一个1个内核线程。优点是可以充分利用CPU的多核能力；缺点是线程数受限于系统对内核线程数的限制，以及上下文切换开销较大

**参考**

> 一文理解JVM线程属于用户态还是内核态: https://cloud.tencent.com/developer/article/1839593

----

## 线程池(ThreadPool)

### 线程池核心线程数的设置

- 原则：`尽可能的不要浪费CPU的计算能力`
- 分析任务类型：CPU计算密集、IO密集、混合型

#### CPU计算密集型任务系统

- 特点：以运算为主，少量的DB等IO开销
- 核心线程数：CPU核数 + 1
- 线程数要尽量少一些，因为CPU在处理计算密集型任务的速度都很快，如果有大量线程在工作，反而会因为线程上下文切换频繁导致CPU性能损耗较大，我们的目的是要充分运用每一个核的算力

#### IO密集型任务系统

- 特点：DB等IO占大头，简单运算为主
- 核心线程数：一般资料建议为2*CPU核数。`实际场景要以性能测试的结果作为参考，监控压测期间的CPU使用率，如果太低，可以持续调大到合理范围`
- 绝大部分与数据库、网络IO交互的任务，都属于IO密集型

**以常见的 controller() -> service() -> dao() 的编码模式举例**
- 由于dao()操作是IO交互，线程不占用CPU运算资源，但是线程仍然是工作态
- 如果核心线程数较小，大部分任务又都在等待队列里无法执行，造成CPU运算资源浪费
- 此时CPU切换线程的开销，远远小于任务等待的性能消耗

#### 混合型任务系统

- 分拆预估任务中的CPU运算时间和IO时间
- 如果相差过大，拆分两个线程池，分别进行CPU密集型预算和IO密集型任务
- 如果相差不大，没必要拆分;各种任务都扔到一个pool中

### 关于应用中设置多少个线程池的思考

- 首先，影响系统性能的不是又多少个线程池，而是`应用有多少线程在工作`

#### 应用实践

- **方案1：** 为CPU计算密集型任务和IO密集型任务分别设置独立的线程池，因为IO密集型任务单个线程执行时间长，CPU利用率又相对较低；如果都在一个线程池中，会影响CPU计算密集型任务抢占CPU快速完成计算

- **方案2：** 对需要快速响应的任务独立设置线程池。如果只有一个线程池，大部分任务会在等待队列中等待，反而会导致需要快速响应的任务响应较慢

- **方案3：** 一类业务逻辑的任务使用独立的线程池

> **RocketMQ的多线程模型举例**
> 
> - 1个Master主线程池（线程数1），负责监听TCP连接，建立连接并创建SocketChannel，注册到selector上（NIO模型）
> - 1个Worker线程池（线程数3），主线程拿到网络数据后提交到Worker线程池
> - 1个DefaultEventExectour线程池（线程数8），将Worker线程池的网络数据进行解编码、SSL校验等操作
> - 1个业务线程池（线程数8），进行实际的业务操作
> 
> RocketMQ的线程模型的原则，首先将线程池按照工作类型进行独立设置；接着根据工作类型是运算密集还是IO密集设置不同的线程数

采用多线程池时，需要注意：不要让每个线程都阻塞在等待未被处理请求的结果上，会造成死锁

### ThreadPoolExecutor参数配置

- corePoolSize: 核心线程数，线程池的实际基础工作能力
- maximumPoolSize： 最大线程数，只有当等待队列满后才会创建
- keepAliveTime： 存活时间，大于核心线程数的线程存活时间；根据实际配置，没有银弹
- unit： 存活时间的时间单位
- workQueue： 等待队列
- threadFactory：线程创建工厂，`建议配置`，为每一个线程设置一个Name。排查问题是，可以通过dump文件较为直观的观察到创建的线程
- RejectedExecutionHandler：线程拒绝策略，当`线程池已达到最大线程数&&等待队列满`时的任务拒绝策略
    - AbortPolicy：默认策略，直接抛出异常`RejectedExecutionException`
    - DiscardPolicy：直接忽略 - 这种策略基本没法使用，无法感知到线程池运行情况
    - CallerRunsPolicy：用当前线程运行任务。问题，会阻塞当前线程的执行
    ```java
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        public CallerRunsPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                // 直接调用了线程的run方法，而不是start
                r.run();
            }
        }
    }
    ```
    - DiscardOldestPolicy：丢弃队列最前面的任务并尝试执行任务
    ```java
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        public DiscardOldestPolicy() { }

        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
    ```

**等待队列的选项思路**

_内存安全方面：_

- LinkedBlockingQueue，要指定长度，否则相当于是无界队列。线程池的工作线程永远都只有corePoolSize；还会造成`内存溢出`
- ArrayBlockingQueue：基于数组实现的等待队列，不会出现LinkedBlockingQueue的问题

_性能方面：_

- LinkedBlockingQueue：使用两把锁控制读写，并发性较高。链表的数据结构特性决定的，FIFO读取的是head头结点并改变指针，进入队列只是指向tail尾结点
- ArrayBlockingQueue：只有一把锁锁住整个数组，并发度较低
- 如果系统是有大量任务被快速提交到线程池的场景，则使用`LinkedBlockingQueue`；一般场景`ArrayBlockingQueue`足以，而且有长度约束，避免内存溢出

### 线程池监控:
- 监控线程池的运行状态，帮助问题排查
- 继承ThreadPoolExecutor重写相关方法

```java
public class MyThreadPool extends ThreadPoolExecutor {
	
    // .... 忽略构造方法 .... //
    
	@Override
	protected void beforeExecute(Thread t, Runnable r) {
		// 线程执行前
	}

	@Override
	protected void afterExecute(Runnable r, Throwable t) {
		// 线程执行后
	}

	@Override
	protected void terminated() {
		// 线程池停止
	}
}
```

### 线程池执行流程总结

**如果当前工作线程数 < 核心线程数：任务直接尝试创建工作线程执行**

- 并发控制基于CAS自旋增加核心线程数计数器
- 核心线程添加失败的并发任务，走`任务直接提交到等待队列进行等待逻辑`

**如果当前工作线程数 >= 核心线程数：任务直接提交到等待队列进行等待**

**如果添加等待队列满，并不是直接执行拒绝策略，而是尝试再创建一个工作线程执行**
- 条件：工作线程数 < 最大线程数
- 创建失败执行拒绝策略

**工作线程被包装`为Worker线程对象，以实现线程的复用`**
- Woker线程被首次创建后，开始调用Runnable的run方法让Worker线程开始工作
- 从等待队列获取任务Runnable对象，并执行run方法(run是同步执行)