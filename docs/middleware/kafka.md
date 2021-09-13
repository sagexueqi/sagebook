# Kafka

## 参数和术语

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

> 

### 生产者参数

### 消费者参数

----

## 编码范式


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
