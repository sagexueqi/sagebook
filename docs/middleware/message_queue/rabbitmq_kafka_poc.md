# RabbitMQ&Kafka Poc

## 工作协议的区别（最主要区别）

- RabbitMQ是`AMQP`协议的实现：使用AMQP协议的消息队列，当消息被投递到消费者后，会将消息在队列中删除，这是破坏性操作（无法回溯）；同时，消费者也无法再重新消费某一个点位的消息（新消费者只能继续消费自己加入后的消息）

- Kafka（以及RocketMQ）是基于`日志`模式的消息队列实现：消息基于日志模式被顺序（`磁盘顺序写入一定快于随机`）的追加到文件中，消费者的消费动作类似于顺序读取日志文件，是只读操作；同时，消费者也可以重置消费点位，消费某一个点位后的消息（新消费者可以从某一个位置继续消费，即可以回溯）

## 实际使用对比

- 消息顺序性：AMQP协议的限制，一个Topic只有一个Queue，如果RabbitMQ实现顺序消息需要单线程发送、单线程消费才可以；Kafka将Topic分为多个Partition，可以实现Partition级别的顺序消息隔离

- 吞吐性能：各种POC显示RabbitMQ的单机QPS在几万，Kafka的单机QPS在百万；仍然是AMQP协议限制，一个Topic只有一个Queue，而Kafka的多Partition提高了Topic的吞吐性能

- 数据可靠性：都支持多副本的Backup

- 可用性方面：Kafka是天然的分布式消息系统（多Partition、多Broker）；RabbitMQ的集群模式（消息仍在一个Master节点，其他Consumer从Slave节点拉取消息时，Master会将消息先发给Slave然后再推给消费者）、RabbitMQ的镜像模式（Slave节点会主动同步Master的所有消息，消费者通过HA-Proxy连接，但是获取消息的时候，还是由Master返回给Slave然后再推送给消费者）

## 技术选型考量

- 一般性场景，可以考虑使用RabbitMQ，编码相对简单，稳定性也足够高，不需要考虑消息丢失和重复消费的问题；高吞吐场景，建议使用Kafka（消息丢失、重复消费、稳定性等问题需要case by case解决）

- 基于公司已有的架构成熟度，如果一直基于Kafka或RabbitMQ作为消息中间件，可以继续采用，降低维护成本与问题处理难度

- 金融系统还是建议优先RabbitMQ（稳定、AMQP也是金融级的协议规范）RabbitMq比kafka成熟，在可用性上，稳定性上，可靠性上，RabbitMq超过kafka

- 吞吐性能两者没有太大可比性，两种中间件的服务目的本就不相同

**参考：**

> AMQP学习 & RabbitMQ 与 ActiveMQ、ZeroMQ以及Kafka的比较: https://www.cnblogs.com/charlesblc/p/6058799.html