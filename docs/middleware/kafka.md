# Kafka

## 关键术语

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

----

## 主要配置参数

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

指定Producer本地内存缓冲区大小。KafKa使用异步发送的消息架构，Producer在启动时会划分一块内存区域缓冲保存待发送的消息，然后由专属线程负责从缓冲区真正读取消息然后发送到Broker。

默认值：`32M`

消息在大量快速生成的过程中，会将buffer内存填满；此时Producer再发消息时会被阻塞，这段时间不能超过`max.blocks.ms`设置的值，一旦超过，producer则会抛出`TimeoutException`异常。

如果说，我们的消息生产者经常报`TimeoutException`异常时，是需要考虑调大`buffer.memory`的配置；由于Producer是线程安全的，多线程共享kafka producer时，很容易把 buffer.memory 打满。

- **max.block.ms**

消息发送阻塞时间。如果消息发送过快导致buffer区被打满，此时Producer会被阻塞，如果超过了`max.block.ms`配置的时间，会抛出`TimeoutException`异常

默认值：`60000`

- **compression.type**

消息压缩类型。有效值为`none、gzip、snappy、lz4或zstd`。Broker会适配压缩配置，压缩的目的在于缩减带宽，但是会降低性能

压缩动作发生在Producer端，Broker不做压缩动作（生产者压缩格式千千万，Broker适配的成本太高，只要可以成功接收、落盘、push出去即可），Consumer做解压。一般消息无需压缩

默认值：`none`

- **retries**

重试次数。失败后自动重试的次数，一直重试到配置值或broker返回ack

默认值：`Integer.MAX_VALUE`

- **batch.size**

批次大小。当消息发送到相同的Partition时，是会被打包成一批发送的，这样可以减少网络交互的开销；

但是这个配置也不能太大，如果太大的话，Producer生产的消息一直在缓冲区发不出去，那消息的延迟就会非常高。

默认值：`16KB`

一般和`linger.ms`搭配使用

- **linger.ms**

批处理延迟时间。很多应用不会频繁大量生成消息，可能需要缓冲区发送到同一个分区的消息才能凑够`batch.size`配置的大小，如果只是简单等到batch.size大小再发送消息，可能很多系统永远都没办法发出消息了；

linger.ms的作用是，即使缓冲区没有凑够batch.size`配置的大小，到时间也要将该分区的消息发送出去，降低消息的延迟

一般和`batch.size`搭配使用，同时设置batch.size和 linger.ms,就是哪个条件先满足就都会将消息发送出去

### 消费者参数

#### fetch.max.bytes

Consumer一次从Broker中拉取消息的最大数据量。

默认值：`50M`

- **max.partition.fetch.bytes**

Consumer一次拉取中，每一个`partition`的最大数据量。

默认值: `1M`

这个参数和`fetch.max.bytes`的区别在于，`fetch.max.bytes`关注一次拉取的所有分区数据量大小之和，而`max.partition.fetch.bytes`关注的是每一个分区的数据量大小

- **fetch.min.bytes**

每次拉取的最小字节数。如果Broker可用的数据量小于 fetch.min.bytes 指定的大小，那么它会等到有足够的可用数据时才把它返回给消费者。这样可以降低消费者和 broker 的工作负载。

和`fetch.max.wait.ms`搭配使用，一般不用配置。

- **fetch.max.wait.ms**

从Broker拉取消息的最大等待时间。同时设置`fetch.min.bytes`和`fetch.max.wait.ms`,哪个条件先满足就都会拉取消息。

默认值：`500`，一般来说无需配置

- **group.id**

消费者组名称。多个消费者实例共同组成的一个组，同时消费多个分区以实现高吞吐。

- **heartbeat.interval.ms**

心跳间隔。与`session.timeout.ms`配合使用

heartbeat线程每隔`heartbeat.interval.ms`时间向`broker-coordinator`发送心跳包，证明还活着。

如果`broker-coordinator`在一个`session.timeout.ms`周期内未收到consumer的心跳，就认为consumer已经挂掉了，会将该消费者移出组，并触发rebalance

默认值：`3000`

- **session.timeout.ms**

会话超时时间。与`heartbeat.interval.ms`配合使用

默认值：`3000`

- **auto.offset.reset**

当Kafka中不存在当前偏移量时的处理逻辑。

取值范围：`latest-从最新offset开始消费`;`earliest-从最早的offset开始消费`

默认值：`latest`

- **enable.auto.commit**

是否开启自动提交。如果开启自动提交，会根据`auto.commit.interval.ms`配置的时间间隔，自动提交当前的offset。

但是开启自动提交，会有重复消费的可能。

默认值：`true`

- **auto.commit.interval.ms**

自动提交时间间隔。与`enable.auto.commit`配合使用，根据配置的时间间隔，提交当前消费的offset

默认值：`5000`

- **max.poll.records 和 max.poll.interval.ms**

max.poll.records：单次消费者拉取的最大数据条数

> poll的过程是从缓冲区拉取，有可能会有网络的`fetch`操作：如果缓冲区为空或`nextInLineRecords`需要fetch，Consumer会从Broker拉取`max.partition.fetch.bytes`限制的数据，这时可能拉取回来1000条
>
> max.poll.records的作用是，调用`kafkaConsumer.poll()`方法，每次从缓冲区拉回多少条数据：例如配置10，那么要把刚才fetch回来的消息全部拿到，需要call 100次poll方法

max.poll.interval.ms：最大拉取时间间隔

> 如果Consumer超过这个时间间隔仍然没有发起poll操作，但是仍然发出心跳；Broker会认为该Consumer处于livelock状态，认为这个Consumer的处理能力太弱，会将其剔除Consumer Group，触发Rebalance操作
>
> Consumer为了自己不被踢出Group，应该不间断的发起poll操作；同时，即使在poll到消息以后，应该尽快提交到工作线程池处理，避免由于消息业务处理时间太长而超时
>
> 不间断发起poll操作的过程，client没有为我们提供实现，需要在应用中自己编码完成

- **partition.assignment.strategy**

分区策略。

RangeAssignor（范围）：该策略会把主题的若干个连续的分区分配给消费者

RoundRobinAssignor（轮询）：该策略把主题的所有分区逐个分配给消费者

StickyAssignor（粘性）：粘性的分区分配策略。该策略会尽可能地保留之前的分配方案，尽量实现分区分配的最小变动。

----

## 编码范式

### 消息生产者

```java
public class Producer {
	private static KafkaProducer<String, String> producer;

	private Producer() {
	}

	private static class InnerKafkaProducer {
		private static final Producer INSTANCE = new Producer();
	}

	/**
	 * 获取单例的Producer
	 *
	 * @return
	 */
	public static Producer getInstance() {
		producer = new KafkaProducer<>(producerConfigs());
		return InnerKafkaProducer.INSTANCE;
	}

	private static Map<String, Object> producerConfigs() {
		Map<String, Object> props = new HashMap<>();
		// SERVER 地址，从环境变量：KAFKA_BOOTSTRAP_SERVERS 中获取
		props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, System.getenv("KAFKA_BOOTSTRAP_SERVERS"));
		// retries 重试次数配置：0表示不配置，默认 Integer.MAX_VALUE = 2147483647，如果消息必须尽最大努力要送达，建议不配置
		props.put(ProducerConfig.RETRIES_CONFIG, 10);
		// batch.size 批次大小：当消息发送到相同的Partition时，是会被打包成一批发送的，默认16K；如果希望消息尽快被发送，可以调整小一些，我们生产配置10K
		props.put(ProducerConfig.BATCH_SIZE_CONFIG, 10240);
		// linger.ms 批处理延迟时间：同一个Partition缓冲区发送消息间隔，一般配置10ms，太久了消息延迟太大
		props.put(ProducerConfig.LINGER_MS_CONFIG, 10);
		// buffer.memory 生产者本地缓冲区大小：默认30M，一般配置64M
		props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864);
		// acks 消息投递确认配置：0-会丢消息 all-吞吐太低 1-较为均衡，master确认接收即可
		props.put(ProducerConfig.ACKS_CONFIG, "1");
		// key.serializer key序列化方式：我们一般使用kafka提供的StringSerializer足以
		props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
		// value.serializer value序列化方式：我们一般使用kafka提供的StringSerializer足以
		props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

		return props;
	}

	/**
	 * 获取KafkaProducer
	 *
	 * @return
	 */
	public KafkaProducer<String, String> producer() {
		return Producer.producer;
	}

	public static void main(String[] args) throws Exception {
		// Producer是线程安全的，多个线程可以共享一个实例
		KafkaProducer<String, String> kafkaProducer = Producer
				.getInstance()
				.producer();

		// 发送消息：直接异步发送，不管结果，有些异常无法捕获 oneway方式
		kafkaProducer.send(
				new ProducerRecord<>("topic_hb_hp_tester", "hello-world!")
		);

		// 发送消息：同步发送，等待发送结果
		try {
			RecordMetadata recordMetadata = kafkaProducer.send(
					new ProducerRecord<>("topic_hb_hp_tester", "test-key", "sync-hello-world!")
			).get();

			System.out.println("----同步发送----");
			System.out.println("topic:" + recordMetadata.topic());
			System.out.println("partition:" + recordMetadata.partition());
			System.out.println("offset:" + recordMetadata.offset());
		} catch (Exception e) {
			// log error
			e.printStackTrace();
		}

		// 发送消息：异步发送，执行回调函数
		kafkaProducer.send(
				new ProducerRecord<>("topic_hb_hp_tester", "test-key", "callback-hello-world"),
				(metadata, exception) -> {
					System.out.println("----异步发送----");
					System.out.println("topic:" + metadata.topic());
					System.out.println("partition:" + metadata.partition());
					System.out.println("offset:" + metadata.offset());
					System.out.println("exception:" + exception);
				}
		);

		// 应用结束前，要close掉producer
		kafkaProducer.close();

		// 为了演示回调，sleep
		Thread.sleep(10000L);
	}
}
```

### 消息消费者

----

## 实际应用

### 容量规划

### 高可用性

----

## 问题排查

### 丢消息与重复投递

----

## 技术选型对比

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
>
> Kafka导致重复消费原因和解决方案: https://blog.csdn.net/lyonliyang/article/details/107310539
