# JVM调优

## 调优的整体思路

- 准备工作: DUMP文件、jmap、jvisualvm
- 原则: 降低Full GC的频率，让对象尽量在年轻代朝生夕死
- 以代码优化为主，jvm参数的调整尽量还是以堆栈内存为主，配合输出gc日志、oom的dump日志等
- 基于dump文件分析内存泄露与线程死锁；通过gc日志分析full gc发生的场景与条件；结合监控平台，分析问题的诱因

```shell
# 打印dump文件
-XX:+HeapDumpOnOutOfMemoryError  
-XX:HeapDumpPath=/usr/local/app/oom

# 输出gc日志
-XX:+PrintGCDetails  -XX:+PrintGCTimeStamps  -XX:+PrintGCDateStamps  -Xloggc:./gc.log
```

## JVM常见内存溢出与解决方案

**java.lang.OutOfMemoryError：Metaspace**-元空间内存溢出，当达到Max上限值并Full GC之后，仍然无法分配空间
- 使用了不合理的JVM参数，导致元空间内存分配过小
- 动态代理没有缓存代理类，而是每次都使用`.getProxy()`生成动态代理类，导致Class类元信息在元空间膨胀
- 使用自定义classloader，重复创建classloader或者重复加载class

**java.lang.OutOfMemoryError: Java heap space**-堆内存溢出
- 常见的内存溢出问题；需要根据Dump文件分析此时堆内存中占比较大的对象，并以此作为排查依据，分析发生问题的代码逻辑；同时要结合GC日志，确认是否是由于堆内存空间不足导致溢出等
- 原因1: 堆内存空间较小，系统又经常创建存活时间较久的大对象，导致新对象无法创建或存活对象无法进入年老代（空间担保机制）
- 原因2: 在循环中大量创建对象，或者在list中add大量对象；表象为dump文件中的大对象一般是应用内的对象
- 原因3: SQL查询没有分页，一次返回大量数据集，年轻代和年老代均无法分配空间；工具、拦截器打印输入输出的日志，输出流对象日志，导致堆内存打满

**java.lang.StackOverflowError**-栈内存溢出
- 原因1: 无限递归，还是要尽量避免递归的使用，即使使用，也要控制好跳出的条件
- 原因2: 循环中大量创建局部变量
- 原因3: 线程栈内存参数值过小，不符合应用的实际场景

## GC常见问题与解决方案
**表象: Young GC、Full GC频繁，一次Full GC后，年老代GC日志显示空间变化非常大**
- 原因分析: 
  - 新对象过早的晋升到年老代，导致Old区有大量本该被回收的垃圾；
  - 过早晋升不会直接导致GC性能降低，而当Eden区空间不足触发Young GC前会要求检查年老代空间或者年老代空间提供担保，此时由于这些垃圾对象的问题，会导致失败，进而引发Full GC
- 解决方案：
  - 检查`-XX:MaxTenuringThreshold`参数配置是否过小，默认为15次YGC晋升到Old区。也可以根据应用特点，适当调大
  - 年轻代或Eden区内存过小`-Xmn`: 导致分配对象时，无法在Eden区完成分配，提前触发YGC，导致对象提前进入Old区；后续分配空间时，YGC前需要Old区提供空间担保失败，进而触发Full GC

**表现: CMS old GC非常频繁(可能在某些监控平台会表现为Full GC频繁)，但是每次GC的耗时不是很久**
- 原因分析: 
  - 代码中有大量的```System.gc()```的显示调用
  - 代码中存在内存泄露，导致Old区有大量被持有的引用,即使Full GC也无法清理；年轻代Young GC前，Old区空间不足或者无法提供担保，触发Full GC 或 针对CMS收集器，会有一个回收阈值参数，当Old区的使用率已经达到阈值，会触发Old GC；
  - 老年代内存过小，导致正常晋升上来的对象增多之后，空间不足，频繁回收垃圾对象
- 解决方案:
  - 通过Code Review、代码扫描等方式，发现代码中的`System.gc()`的调用，评估后移除；如果想 help gc，可以在方法结束后将变量赋值为null，减速gc回收对象
  - 分析代码中的内存泄露对象:
    - 摘掉应用流量，观察GC日志，在cms回收前后分别dump一次。也可以在GC前dump一份，之后使用`jmap -histo:live`触发full gc后打dump文件
    - 将dump文件导入到MAT/JVISUALVM等工具中，分析大对象的占比，着重关注应用领域内的类
    - 寻找大对象生成的方法、入口，review代码逻辑，是否有对象没有释放、持续持有的情况
    - fix bug。发布后观察gc日志
  - 在没有泄露的情况下，增大堆内存空间，持续观察

**表现: 单次CMS Old GC(或者Full GC)的耗时非常长，而且是偶发现象**
- 原因分析：
  - Full GC垃圾收集器退化为`Serial Old`收集器: 
    - 在Young GC后会将对象晋升到老年代失败(没有足够连续空间)，导致发生`promotion failed 晋升失败`现象，正常来说CMS会对old进行垃圾回收；
    - 如果此时有CMS在回收垃圾，会产生`concurrent mode failure 收集器无法处理浮动垃圾`问题，收集器退化为`Serial Old`导致耗时增加
    - 确认GC日志中是否有上述关键字
- 解决方案:
  - 让CMS收集器，在回收指定次数年老代后，整理内存空间，减少碎片: `-XX:CMSFullGCsBeforeCompaction`
  - 堆内存空间过大，导致正常Full GC的时间过久，合理控制堆内存大小
  - 提早触发CMS Old GC，预留年老代空间，修改参数`-XX:CMSInitiatingOccupancyFraction=N`设定CMS在对内存占用率达到N%的时候开始GC

**参考**

> 来一道 PerfMa 面试必考的 GC 题: https://www.codercto.com/a/24558.html  
>
> Java中9种常见的CMS GC问题分析与解决: https://tech.meituan.com/2020/11/12/java-9-cms-gc.html 
>
> 老大难的GC原理及调优，这下全说清楚了: https://juejin.cn/post/6844903953004494856 
>
> CMS收集器产生的问题和解决方案: https://www.cnblogs.com/hongdada/p/10489643.html#%E6%8F%90%E5%8D%87%E5%A4%B1%E8%B4%A5%EF%BC%88promotion-failed%EF%BC%89