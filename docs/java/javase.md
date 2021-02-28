# JavaSE

## equals()和hashCode()

  一般来说，在使用HashMap或Set进行去重的场景，需要重写equals和hashCode方法

  hashmap首先根据hashCode判断两个对象是否相等，如果不相等直接跳过做put后续操作；如果相等，就执行equals继续比较，如果相等就不做任何put操作，直接跳过

  **最佳实践：**

  - hashCode使用唯一字段生成，equals进行唯一字段的比较
  - 参数hashCode计算的字段，不能修改

----

## 集合

### HashSet
- 无需不重复集合
- 基于HashMap实现，可以理解为value为null的HashMap
- 没有get()方法
- 线程不安全

----

### HashMap
**K-V键值对，通过KEY的hashCode计算index位置**

**数据结构**: 数组(bucket) + (链表 + 红黑树)`解决hash冲突`
- 数组：存放hash value的桶
- 链表：jdk1.8采用尾插法，解决jdk1.7之前在扩容时hash碰撞导致的死循环问题
- 红黑树：当链表长度=`6`时进化为红黑树；当节点数小于`8`时，退化为链表
  - 进化为红黑树的目的在于快速查找，红黑树遍历的时间复杂度为：`O(logN)`;而链表遍历的时间复杂度为：`O(N)`-因为尾插法，最新的记录在尾部
  - 为什么小于8时退化，这是避免频繁的在链表和红黑树间变化

### ConcurrentHashMap
- `get()`方法时无锁的
- `put()`操作时，如果bucket为null，基于`CAS`无锁的方式设值value；如果bucket已经有值或者CAS失败，对当前Node加synchronized锁同步

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V> implements ConcurrentMap<K,V>, Serializable {
    final V putVal(K key, V value, boolean onlyIfAbsent) {    
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // 当前位置为null，基于CAS赋值，如果此时hash冲突，进入下一次循环
            // Node<K,V> f 不为空，进入到synchronized (f)逻辑，加锁赋值
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    // 尾插法解决hash冲突
                }
            }
        }
        return null;
    }

}
```
----

### ArrayList
- 基于数组实现
- 尾部元素插入、删除快，随机查找(指定位置的查找)速度快
- 随机查找时间复杂度: `O(1)`; 遍历时间复杂度: `O(n)`

### LinkedList
- 链表实现
- 元素随机插入、删除快，随机查找速度慢
  - 针对插入和删除，只需要修改索引位置前后的指针即可
  - 查找需要从头遍历
- 随机查找时间复杂度: `O(n)`; 其余操作均为: `O(1)`

### CopyOnWriteArrayList
- 线程安全的ArrayList
- 基于读写分离与最终一致性模型
- 修改时，基于`ReentrantLock`加锁，copy一份副本进行修改，最后再重新指向
- 读取时，无锁，直接获取当前Array进行读取

----

## 多线程

### 实现线程的几种方式（thread  runnable  callable）

- Thread: 继承Thread类，重写run方法，调用start方法启动线程
- Runnable：实现Runnable接口，重写run方法，作为new Thread的参数，调用start方法启动线程
- 直接调用run: 同步执行
- Callable: Future/Task模式，submit线程池，等待获取所有线程消费结果
    - 异常如何感知：封装Result对象，call逻辑中catch异常，写入Result中

### 线程锁（synchronized Lock）
- synchronized: JVM级别，可重入，修饰方法或代码块
    - 修饰代码块: 锁住同步对象，例如class、static对象
  
- Lock: API级别，基于AQS实现，可重入，例如`ReentrenLock`
    - 默认非公平、基于AQS的state记录重入次数、CAS设置等待列表结点
    - 读写锁：读读不冲突、读写冲突、写写冲突。`state`的高低位标示读写
  
- 性能差别不大，还是看使用场景。如果希望锁粒度更小，更可控，推荐使用Lock

### countdownlatch cyclicbarrier   semaphore volatile等关键字的用法

- semaphore: 信号量
    - 使用场景：并发控制，限流；限制连接池获取资源的数量
    - 基于AQS实现。state是许可数量
    - **作为限流方案的问题：**无法平滑瞬时流量，许可可以在很短的时间内被消费，如果不归还无法继续获取

- countdownlatch: 做减法，不能重复使用

- cyclicbarrier: 做加法，可以重复使用，但是必须reset归0。没有信号量灵活

- volatile：保证可见性和避免JVM的指令重排，并不能保证线程安全。
    - 根据Java内存模型，所有变量的操作都是在当前线程的缓存中操作，而使用volatile关键字修饰的属性，会直接在主工作内存操作

----

### 线程池

- 核心参数：coreSize-核心线程数、maxSize-最大线程数、keepAliveTime-存活时间、unit-单位、queue-等待队列、factory-线程工厂、rejectHandle-拒绝策略

- 等待队列的选择：
```
    - ArrayBlockingQueue：有界队列
    - LinkedBlockingQueue：默认无界队列。对于选择LinkedBlockingQueue时，最大线程数参数无效
```

- 拒绝策略：
```
    - 直接拒绝并抛出异常
    - 直接丢弃
    - 尝试在当前线程执行任务
    - 尝试丢弃队列头任务
```

- execute()工作流程：   
```
    - 如果当前工作线程数 < 核心线程数：直接创建Worker执行当前线程任务
    - 如果当前工作线程数 >= 核心线程数：加入到等待队列中
    - 如果等待队列满：当前线程池中的线程数没有大于最大线程数，创建新线程执行当前任务。否则执行拒绝策略
```

- 等待队列的任务是如何被消费的？
    - 线程池中的工作线程是包装为Worker线程，执行任务线程的run方法，而不是start方法
    - 由于提交任务而创建的Worker线程工作完成后，会循环从队列中获取任务执行
