# JavaSE

## 集合Set、List、Map以及线程安全问题

### HashSet
- 无需不重复集合
- 基于HashMap实现，可以理解为value为null的HashMap
- 没有get()方法
- 线程不安全

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

## equals()和hashCode()

  一般来说，在使用HashMap或Set进行去重的场景，需要重写equals和hashCode方法

  hashmap首先根据hashCode判断两个对象是否相等，如果不相等直接跳过做put后续操作；如果相等，就执行equals继续比较，如果相等就不做任何put操作，直接跳过

  **最佳实践：**

  - hashCode使用唯一字段生成，equals进行唯一字段的比较
  - 参数hashCode计算的字段，不能修改

----