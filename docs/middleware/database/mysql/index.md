# Index - 索引

## 索引的数据结构

### Hash索引

哈希索引基于哈希表实现，只有精确匹配索引的所有列的查询才有效。对于每一行数据，存储引擎都会对所有的索引列计算一个哈希码，哈希码是一个较小的值，并且不同键值的行计算出来的哈希码也不一样。哈希索引将所有的哈希码存储在索引中，同时在哈希表中保存指向每个数据行的指针。

**优点：** 精确单行匹配速度快，Hash算法时间复杂度`O(1)`

**缺点：**

- 用于排序、遍历，时间复杂度退化为O(n)
- Hash冲突时，维护代价高
- InnoDB不支持哈希索引

### 二叉树索引