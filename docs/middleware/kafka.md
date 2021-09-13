# Kafka

## 术语和参数

**Broker**

Kafka的服务器端是由被称为Broker的进程组成，一个Kafka集群是由多个Broker组成。Broker负责接收客户端发送的请求，并对消息进行持久化。虽然多个Broker可以运行在一台服务器上，但更常见的做法是将不同的 Broker 分散运行在不同的机器上，这样如果集群中某一台机器宕机，即使在它上面运行的所有 Broker 进程都挂掉了，其他机器上的 Broker 也依然能够对外提供服务。这其实就是 Kafka 提供高可用的手段之一。

**Topic**

发布订阅消息的对象主体是`主题`，你可以为每个业务、每个应用甚至是每类数据都创建专属的主题。

**Producer**

消息生产者。发布消息的客户端应用成为生产者。

**Consumer**

消息消费者。订阅这些主题消息的客户端应用程序就被称为消费者。

**Partition**

分区。在Kafka中，一个Topic被分配为多个分区(`Partition`)，每个分区是一组有序的消息日志。生产者在某一个Topic上生产的消息，只会被发送到该Topic的其中一个Partition中。这也就是说，一条消息不是在Partition-0，就是在Partition-1。

> 为什么要有Partition的概念？
>
> - kafka中，Topic是一个逻辑上的概念，而Partition是物理上的概念。生产者和消费者只对Topic感兴趣，至于Topic在Broker上是如何组织存储的，是不关心的。
>
> - 出于性能考虑，如果一个Topic的数据都存在一个Broker或者队列中，那么这个Broker或者队列就成为了性能和可用性的瓶颈。而基于Partition的概念，一个Topic的存储就可以做到横向的扩展。
>
> - 在物理上，一个Partition对应一个物理文件夹，我们可以认为消息在文件中是顺序写入的（磁盘顺序写入的性能是远大于随机写入的，不论是机械硬盘还是SSD硬盘）；而一个Broker中，一个Topic可以存放多个Partition文件
>
> - 这样，Producer在生产消息时，可以将消息负载分发到某一个Broker中的某一个Partition中

**Segment**

段。Partition在物理上是由多个`Segment-段`组成的；在Broker的物流磁盘中，Partition是一个文件夹，而消息是按照时间顺序存储在一个一个的Segment文件中。

> 既然已经有了Partition分区概念，为什么还需要Segment？
>
> - 如果不引入Segment的概念，一个Partition的消息都在同一个文件中，随着时间的推移，这个文件会越来越大，必然需要处理文件压缩的问题，否则空间会被无线浪费
>
> - 消息在Partition中按照Partition组织后，当Kafka进行数据压缩时，只需要将旧Segment文件进行删除即可，并不会影响到新Segment文件的顺序写入

**Consumer Group**

消费者组。指的是多个消费者实例共同组成一个组(groupId)来消费一组`Topic-主题`。Topic的每个`Partition-分区`在同一个`Consumer Group-消费者组`中，只能有一个`Consumer`；但是，这一个消费者可以消费多个分区。

同时，同一个消费者组内的消费者实例不要大于Partition的数量，否则多余节点是不会消费消息的。

> 为什么一个Partition在同一个Group中只能有一个Consumer？
>
> - Partition是最小单位，如果说同一个Group内的多个Consumer同时消费Message的话，由于每一个节点的处理速度一定是不一样的，那必须等待所有的消费者消费完消息后，才可以推送（或者叫被拉取）下一条消息，否则同一个组内的消费就不是`顺序`和同步的
>
> - 我们可以认为不同的消费者组，有不同的消费逻辑（即使是同样的代码）；那只需要维持每一个消费者组已经消费的offset即可，这也降低了复杂性
>
> - 关键还是为了保证消费消息时的顺序性
>
> 如果我们希望应用的所有节点，都可以消费到Topic中的消息怎么办？
>
> - 应用启动时，每一个节点随机生成一个唯一的消费者组名称

![kafka_partition_group](./imgs/kafka_partition_group.jpg)

**Consumer Offset**

偏移量。每个消费者在消费Partition的消息时，一定有一个字段记录已经消费到哪一个位置，这个字段就是消费者位移（Consumer Offset）。注意，这个和分区位移是不同的。

分区位移是消息在Partition的位置，一旦消息被成功的写在某一个Partition上后，这个值就是固定不变的。

而消费者位移则不同，它是消费者消费进度的指示器嘛，不同消费者的位移是不同的。可以理解为书签。

**Replication**

副本。副本是把相同的数据拷贝到多台机器上来实现高可用。分为领导者副本和跟随者副本。

领导者对Producer和Consumer提供服务，而Replication只从Leader接受数据同步。和Raft中的Leader、Replication关系一致，Replication即使接收到Consumer的请求，也会将其转发到Leader节点。

**Rebalance**

重平衡。这是Kafka实现高可用的手段之一，当一个Consumer节点挂掉之后，kafka会检测到并将挂掉实例负责的分区，重新分配给其他消费者。

### Broker参数

- **auto.create.topics.enable**

是否允许自动创建 Topic。开发测试环境允许自动创建，生产环境应该由运维根据工单操作。

- **message.max.bytes**

Broker 端对 Producer 每批发送过来的消息也有一定的大小限制。

默认值：`1M`

- **replica.fetch.max.bytes**

Broker可复制的最大字节数。要大于`message.max.bytes`的配置，否则Master节点可以接受消息，但是无法复制到Replication节点，从而造成数据丢失。

- **log.retention.hours/log.retention.minutes/log.retention.ms**

消息保存时长。个参数可以同时设置，kafka会优先使用最小值的参数，kafka默认log.retention.hours=168， topic具有相同的参数，会覆盖调broker配置。

### 生产者参数

- **acks**

broker的确认数，通常有`0、1、all`三种常见配置。

0：生产者完全不等待Broker的响应，记录添加到Producer缓冲区后就视为发送成功。这种配置，生产者吞吐量最高，但是不能保证消息被成功投递；适合非重要消息、oneway消息；

1：生产者同步等待Broker主节点已经成功确认消息，即视为发送成功。不会等待Replication节点的同步结果。这种配置最平衡，只要主节点刷盘成功，消息就不会丢失；只有主节点没有刷盘，同时也没有同步到Replication节点，宕机恢复后，消息会丢失；

all：表示领导者和跟随者都确认成功，才视为已发送。效率是三者里面最低的。如果要确保不丢消息就要设置为all

- **buffer.memory**

### 消费者参数

----

## 编码范式

----

### 生产使用经验

#### 容量规划

#### 问题排查

#### 高可用性

#### 丢消息与重复投递

**参考：**
> Kafka中文文档：https://kafka.apachecn.org/
>
> Kafka万亿级消息实战：https://cloud.tencent.com/developer/article/1825906
>
> kafka_pro: https://github.com/youyangkou/kafka_pro
>
> kafka学习非常详细的经典教程: https://blog.csdn.net/tangdong3415/article/details/53432166
>
> kafka存储机制: https://www.cnblogs.com/cynchanpin/p/7339537.html
