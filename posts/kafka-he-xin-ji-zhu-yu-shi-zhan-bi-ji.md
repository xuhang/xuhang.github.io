---
title: 'Kafka 核心技术与实战笔记'
date: 2024-12-08 11:01:07
tags: [kafka,geektime,note]
published: true
hideInList: false
feature: 
isTop: false
---
## 术语

发布订阅的对象是==主题（Topic）==，向主题发布消息的客户端应用程序称为==生产者（Producer）==，订阅主题消息的客户端应用程序称为==消费者（Consumer）==，生产者和消费者统称为==客户端（Clients）==。

Kafka 的服务器端由被称为 ==Broker== 的==服务进程==构成，即一个 Kafka 集群由多个 Broker 组成，Broker 负责接收和处理客户端发送过来的请求，以及对消息进行持久化。虽然多个Broker可以同时运行在一台机器上，但是通常将不同的Broker分散在不同机器上，以保证高可用。

实现高可用的另一个手段就是==备份机制（Replication）==。备份的思想很简单，就是把相同的数据拷贝到多台机器上，而这些相同的数据拷贝在 Kafka 中被称为==副本（Replica）==。副本的数量是可以配置的，这些副本保存着相同的数据，但却有不同的角色和作用。Kafka 定义了两类副本：==领导者副本（Leader Replica）==和==追随者副本（Follower Replica）==。前者对外提供服务，这里的对外指的是与客户端程序进行交互；而后者只是被动地追随领导者副本而已，不能与外界进行交互。

<!-- more -->


副本机制可以保证数据的持久化或消息不丢失，但没有解决伸缩性的问题（Scalability），如果领导者副本积累了太多的数据以至于单台 Broker 机器都无法容纳了，此时就需要把数据分割成多份保存在不同的 Broker 上。这种机制就是所谓的==分区（Partitioning）==。

Kafka 中的分区机制指的是将每个主题划分成多个==分区（Partition）==，每个分区是一组==有序==的消息日志，生产者生产的每条消息只会被发送到一个分区中。

**副本是在分区这个层级定义的。**每个分区下可以配置若干个副本，其中只能有 1 个领导者副本和 N-1 个追随者副本。生产者向分区写入消息，每条消息在分区中的位置信息由一个叫==位移（Offset）==的数据来表征。分区位移总是从 0 开始，单调递增且不会改变。

Kafka 使用==消息日志（Log）==来保存数据，一个日志就是磁盘上一个==**只能追加写**（Append-only）==消息的物理文件。因为只能追加写入，故避免了缓慢的随机 I/O 操作，改为性能较好的顺序I/O 写操作，这也是实现 Kafka 高吞吐量特性的一个重要手段。不过如果你不停地向一个日志写入消息，最终也会耗尽所有的磁盘空间，因此 Kafka 必然要定期地删除消息以回收磁盘。怎么删除呢？简单来说就是通过==日志段（Log Segment）==机制。在 Kafka 底层，一个日志又近一步细分成多个日志段，消息被追加写到当前最新的日志段中，当写满了一个日志段后，Kafka 会自动切分出一个新的日志段，并将老的日志段封存起来。Kafka 在后台还有定时任务会定期地检查老的日志段是否能够被删除，从而实现回收磁盘空间的目的。

==消费者组（Consumer Group）==指的是多个消费者实例共同组成一个组来消费一个主题，主题中的每个分区只会被组内的一个消费者实例消费。这样可以有效提升消费端的吞吐量。此外，Kafka如果检测到消费者组内某个实例挂了，会把这个实例之前负责的分区转移给其他消费者，这个过程就是==重平衡（Rebalance）==。

![](https://xuhang.github.io/post-images/1733626924845.png)

当数据在网络和磁盘进行传输时，避免昂贵的内核态数据拷贝，Kafka使用了Linux平台实现的==零拷贝（Zero Copy）==技术。

Kafka版本演进

+ 0.8 引入副本机制，至此成为一个真正意义上完备的分布式高可靠消息队列解决方案，此版本API需要指定ZooKeeper地址；
+ 0.8.2.0 社区版引入新版本Producer API，此版本需要指定Broker地址；
+ 0.9.0.0 增加基础的安全认证/权限，使用Java重写消费者API，引入Kafka Connect组件，此版本Producer API比较稳定；
+ 0.10.0.0 里程碑，引入Kafka Stream；
+ 0.10.2.2 新版本Consumer API，修复了可能导致Producer API性能降低的bug；
+ 0.11.0.0 提供幂等性Producer API和事务 API，对消息格式做了重构。

## 集群参数配置

**Broker端参数**

```properties
log.dirs: Broker需要使用的文件目录路径，逗号分割，最好是不同物理磁盘上的目录，能提高吞吐量
log.dir: Broker需要使用的单个文件路径
zookeeper.connect: zk地址
# chroot zk别名，加在zookeeper.connect参数最后，不用每个zk地址都加
# zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka1
listeners: 协议名称://主机:端口, 协议名称://主机:端口... 告诉外部连接着通过什么协议访问主机名和端口开放的kafka
advertised.listeners: 这组监听器是Broker用于对外发布的（通常是双网卡的外网IP）
# 协议可以自定义，如果自定义协议名称，必须用listener.security.protocol.map参数指定具体底层安全协议
listener.security.protocol.map=CONTROLLER:PLAINTEXT :表示CONTROLLER自定义协议底层使用明文不加密传输
auto.create.topics.enable: 是否允许自动创建Topic
unclean.leader.election.enable: 是否允许Unclean Leader（落后很多的分区副本）选举
auto.leader.rebalance.enable: 是否允许定期进行Leader选举
log.retention.{hour|minutes|ms}: 控制一条消息数据被保存的时间
log.retention.bytes: Broker为消息保存的总磁盘容量大小
message.max.bytes: 控制Broker能够接收的最大消息大小
```

**Topic级别参数**

> Topic 级别参数会覆盖全局Broker 参数的值

```properties
retention.ms: 该Topic消息被保存的时长，默认7days
retention.bytes: 为该Topic预留的磁盘空间，默认-1，无限制
max.message.bytes: Broker能够正常接收该Topic的最大消息大小
```

**JVM参数**

```properties
KAFKA_HEAP_OPTS: 堆大小
KAFKA_JVM_PERFORMANCE_OPTS: GC参数
```

**操作系统参数**

```shell
ulimit -n 1000000: Linux操作系统能打开的最大文件描述符数量，尽可能大
```

操作系统其他参数包括swap调优和Flush落盘时间，swap如果设置为0，当物理内存耗尽，操作系统会随机选择一个进程 kill 掉，无法观测到 Broker 性能的下降和预警。Kafka只要将数据写入到操作系统的页缓存就可以任务消息写入成功，之后由操作系统定期将脏数据落盘到磁盘上。鉴于Kafka提供的多副本冗余机制，可以稍微将定期刷盘的间隔增大。

## 分区机制

**分区策略**（由生产者决定消息发送到主题的哪个分区）

+ 自定义分区策略：通过生产者端`partitioner.class`参数指定分区策略，需要指定一个实现`org.apache.kafka.clients.producer.Partitioner`接口的类并实现其中的`partition()`方法。
+ 轮询策略（Round-robin）：顺序分配，默认分区策略；
+ 随机策略（Randomness）：随机地将消息放到某个分区中；
+ 按消息键保序策略（Key-ordering）：同一个key的所有消息进入相同的分区

## 压缩

压缩可能发生在两个地方：生产者和Broker端

生产者程序中配置`compression.type`启用指定的压缩算法类型，比如

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("acks", "all");
props.put("key.serializer", "org.apache.kafka.common.seriailization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
prosp.put("comression.type", "gzip"); // 启用GZIP压缩
Producer<String, String> producer = new KafkaProducer<>(props);
```

大部分情况下，Broker端接收到Producer的消息后原封不动地保存，但是有两种例外情况可能让Broker重新压缩消息：

1. Broker端指定了和Producer端不同的压缩算法。Broker端接收到消息后先用Producer端的压缩算法解压，然后用Broker端的压缩算法重新压缩。
2. Broker端发生了消息格式转换。主要是为了兼容老版本的消费者程序。

Kafka会将消息使用的压缩算法封装进消息集合中，当Consumer读取消息集合时，就知道使用哪种压缩算法解压了。Broker端也会进行解压缩，目的时为了对消息执行验证，但是会对Broker端性能造成一定影响。

Facebook Zstandard提供的各压缩算法benckmark结果

| 算法            | 压缩比 | 压缩吞吐量 MB/s | 解压吞吐量 MB/s |
| --------------- | ------ | --------------- | --------------- |
| zstd 1.3.4-1    | 2.877  | 470             | 1380            |
| zlib 1.2.11-1   | 2.743  | 110             | 400             |
| brotli 1.0.2-0  | 2.701  | 410             | 430             |
| quicklz 1.5.0-1 | 2.238  | 550             | 710             |
| lzo1x2.09-1     | 2.108  | 650             | 830             |
| lz4 1.8.1       | 2.101  | 750             | 3700            |
| snappy 1.1.4    | 2.091  | 530             | 1800            |
| lzf 3.6-1       | 2.077  | 400             | 860             |

## 无消息丢失配置

当Kafka的若干个（可以选择一个或者所有）Broker成功接收到一条消息并成功写入到日志文件后，会告诉生产者这条消息已成功提交。对于“已提交”的消息，Kafka会保证消息有限度的持久化。

如果生产者使用的是`producer.send(msg)`发送消息，这个API是异步的，调用后立即返回，无法知道消息是否发送成功。应该使用`producer.send(msg, callback)`，这个API能准确的告诉客户端消息是否提交成功。

`acks=all`表示Producer对“已提交”消息的定义，all表明需要所有副本Broker都接收到消息才算“已提交”。

`retries=3`表示Producer的自动重试消息发送次数。

## 拦截器

生产者拦截器，在消息发送前和消息成功提交后插入逻辑；消费者拦截器，在消息消费前和提交位移（offset）后插入逻辑。

自定义生产者拦截器需要实现`org.apache.kafka.clients.producer.ProducerInterceptor`接口，核心方法：

1. `onSend`：在消息发送之前被调用
2. `onAcknowledgement`：在消息成功提交或发送失败后被调用，`onAcknowledgement`的调用要早于生产者发送消息的回调callback的调用

自定义消费者拦截器需要实现`org.apache.kafka.clients.consumer.ConsumerInterceptor`接口，核心方法：

1. `onConsume`：在消息被Consumer处理之前调用
2. `onCommit`：Consumer在提交offset之后调用

## 连接

在==**创建**== KafkaProducer 实例时，生产者应用会在后台创建并启动一个名为 Sender 的线程，该 Sender 线程开始运行时首先会创建与 Broker 的连接。也就是说，在调用send方法前，Producer就已经连接到Broker了，连接的是`bootstrap.servers`指定的Broker地址。实际使用中，并不需要把所有Broker地址都配置在`bootstrap.servers`参数中，因为为 Producer 一旦连接到集群中的任一台Broker，会向某一台Broker发送METADATA请求，就能拿到整个集群的 Broker 信息。

当 Producer 更新了集群的元数据信息之后，如果发现与某些 Broker 当前没有连接，那么它就会创建一个 TCP 连接。同样地，当要发送消息时，Producer 发现尚不存在与目标 Broker 的连接，也会创建一个。

Producer端参数`connections.max.idle.ms`配置了Kafka关闭TCP连接的空闲时间，默认9分钟。如果设置为 `-1`则表示不关闭空闲的TCP连接。

和生产者不同的是，消费者端构建`KafkaConsumer`实例时是不会创建任何TCP连接的，而是在调用`KafkaConsumer.poll`方法时创建。再具体一点，poll方法内部有3个创建TCP连接的时机：

1. 发起`FindCoordinator`请求时。消费者向集群中负载最小（待发送请求最少）的Broker发送请求，查询Coordinator对于的Broker，这一步会创建Socket连接。这个TCP之后会被关闭。
2. 完成上一步的`FindCoordinator`请求后，连接正真的Coordinator，此时会创建Socket连接。
3. 消费数据时。消费者会为每个要消费的分区创建与该分区Leader副本所在的Broker连接的TCP。

消费者关闭Socket也分为主动关闭和Kafka自动关闭。主动关闭是调用API关闭，如`KafkaCosnumer.close()`，自动关闭由消费者端参数`connection.max.idle.ms`控制，默认9分钟。

## 消息可靠性保障

Kafka消息交付可靠性保障：

1. ==最多一次==：消息可能会丢失，但不会重复发送（禁止Producer重试）
2. ==至少一次==：消息不会丢失，但可能重复发送（Kafka默认）
3. ==精确一次==：消息不会丢失，也不会重复发送（幂等和事务）

**幂等性（Idempotence）**

指的是某些操作或函数能够被执行多次，但每次得到的结果都是不变的。

0.11.0.0之后引入了幂等性Producer，开启方法：

```java
props.put("enable.idempotence", true);
// or
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
```

为实现幂等性，Kafka底层设计架构中引入了ProducerID和SequenceNumber

- ProducerID：在每个新的Producer初始化时，会被分配一个唯一的ProducerID，这个ProducerID对客户端使用者是不可见的。
- SequenceNumber：对于每个ProducerID，Producer发送数据的每个Topic和Partition都对应一个从0开始单调递增的SequenceNumber值。

Producer在每条消息中附带了PID（ProducerID）和SequenceNumber。Broker 为每个 Topic 的每个 Partition 都维护了一个当前写成功的消息的最大 PID-SequenceNumber 元组，当 Broker 收到一个比当前最大 PID-Sequence Number 元组小（或等于）的 SequenceNumber 消息时，就会丢弃该消息，以避免造成数据重复存储。

**幂等性Producer** 

能够保证某个主题的**一个分区上**不出现重复消息，它无法实现多个分区的幂等性。或者当Producer进程重启后，这种幂等性也会丢失。

要实现多分区及多会话上的消息不重复，需要事务（Transaction）或依赖事务型Producer。

**事务（Transaction）**

主要是在 read committed 隔离级别上做事情。它能保证多条消息原子性地写入到目标分区，同时也能保证 Consumer 只能看到事务成功提交的消息。

**事务型 Producer**

事务型 Producer 能够保证将消息==原子性==地写入到多个分区中。这批消息要么全部写入成功，要么全部失败。另外，事务型 Producer 也不惧进程的重启。Producer 重启回来后，Kafka 依然保证它们发送消息的精确一次处理。

```properties
// 开启事务型Producer
enable.idempotence=true
```

在Consumer端读取事务型Producer发送的消息，设置 `isolation.level`参数：

1. `read_uncommitted`：默认值，Consumer能读取Kafka写入的任何消息
2. `read_committed`：Consumer只会读取事务型Producer成功提交的事务写入的消息，也能看到非事务型Producer写入的所有消息。

## 消费者组

Consumer Group 是 Kafka 提供的可扩展且具有容错性的消费者机制，组内的多个消费者或消费者实例共享一个公共的ID（Group ID），组内所有消费者协调在一起消费订阅主题的所有分区。

如果所有实例都属于同一个 Group，那么它实现的就是消息队列模型；如果所有实例分别属于不同的 Group，那么它实现的就是发布 / 订阅模型。·

理想情况下，Consumer 实例的数量应该等于该 Group 订阅主题的分区总数。

在新版本的 Consumer Group 中，Kafka 社区重新设计了 Consumer Group 的位移管理方式，采用了将位移保存在 Kafka 内部主题（ __consumer_offsets）的方法。

Rebalance 本质上是一种协议，规定了一个 Consumer Group 下的所有 Consumer 如何达成一致，来分配订阅Topic 的每个分区。

触发Rebalance的3个条件：

1. 组成员数发生变更。有新的Consumer加入或有Consumer崩溃下线
2. 订阅主题数发生变更。Consumer Group 可以使用正则表达式的方式订阅主题，比如`consumer.subscribe(Pattern.compile(“t.*c”)) `，当有新的主题匹配正则表达式则会触发
3. 订阅主题的分区数发生变更。

每个Consumer会定期地向Coordinator发送心跳，如果Consumer不能及时地发送心跳请求，Coordinator就会将其从Group中移除，开启新一轮的Rebalance。这个时间由参数`session.timeout.ms`控制，默认10s。参数`heartbeat.interval.ms`控制心跳请求的频率，Coordinator通过心跳请求的响应通知Consumer开启Rebalance。参数`max.poll.interval.ms`参数限定Consumer端两次poll调用的最大间隔，默认5min，如果超过这个时间，表明Consumer无法顺利地消费完poll的消息，Consumer会主动离开Group，Coordinator会开启新一轮的Rebalance。

但是，Rebalance发生时，所有Consumer实例都会停止消费，直到Rebalance完成。Rebalance过程中，由协调者Coordinator组件帮助完成订阅主题分区的分配，Coordinator专门为Consumer Group 服务，负责为Group执行Rebalance以及提供位移管理和组成员管理等。所有Broker在启动时，都会创建和开启相应的Coordinator组件。

**协调者Coordinator**

协调者专门为Consumer Group 服务，负责为Group执行Rebalance以及提供位移管理和组成员管理。Consumer在提交位移时，其实是向Coordinator所在的Broker提交位移；当Consumer启动时，也是向Coordinator所在的Broker发送各种请求，然后由Coordinator执行消费者组的注册、成员管理等元数据管理。

所有的Broker都有各自的Coordinator组件。

1. 通过计算`partitionId = Math.abs(groupId.hashCode() % offsetTopicPartitionCount)`确定由位移主题的哪个分区来保存Group的数据。
2. 找到该分区的Leader副本，该Broker就是对于的Coordinator。

**位移主题 Offset Topic**

`__consumer_offsets`使用Kafka自定义的消息格式，主题中的key包含3部分内容：GroupID，主题名，分区号。

主题消息体的格式：

1. 包含位移数据、时间戳和自定义的数据
2. 用于保存Consumer Group信息的消息
3. 用于删除Group过期位移甚至是删除Group的消息（消息体为null，当某个Consumer Group下所有Consumer都停止且位移数据都已被删除，Kafka会向位移主题的对应分区写入）

当Kafka集群中第一个Consumer启动时，会自动创建位移主题。`__consumer_offsets`主题的分区数通过Broker端参数`offset.topic.num.partitions`设置，默认50，副本数默认3。

如果Consumer采用的是自动提交位移的方式，就会无限期地向`__consumer_offsets`中写入主题消息。此时需要Kafka定期删除过期消息。Kafka使用的是`Compact`策略，即对于主题中同一个Key的多条消息，只保留最新的消息。Kafka提供了专门的后台线程（`Log Cleaner`）定期巡检待Compact的主题，看看是否存在满足条件的可删除数据。

## 多线程消费者实例

`KafkaConsumer`是单线程设计的（心跳线程是单独的线程），能简化Consumer端的设计，客户端可以在Consumer获取到消息后，多线程处理消息。

`KafkaConsumer`类不是线程安全的，如果在多线程中共享一个`KafkaConsumer`实例，程序会抛出异常。可以多线程中每个线程维护一个专属的`KafkaConsumer`实例，负责完整的消息获取、处理流程。

## 消费进度

Kafka监控的是分区的Lag，如果要计算主题的Lag，需要手动汇总素有分区的Lag。如果消费者速度赶不上生产者的速度，会导致数据不在操作系统的页缓存中，而是磁盘中，这样这些数据就不能通过零拷贝传输，进一步拉大消费者和生产者的差距，Lag越来越大。

有3种方法监控消费进度：

1. Kafka自带的命令行工具`kafka-consumer-groups`脚本
2. Kafka Java Consumer API编程
3. Kafka自带的JMX监控指标

## 副本

副本机制，也称备份机制，通常指分布式系统在多台机器上有相同的数据拷贝。

1. 提供数据冗余。即使部分组件失效，系统依然能继续运转，提高整体可用性和数据持久性。
2. 提高高伸缩性。支持横向扩展，能通过增加机器的方式提高读写性能，进而提高吞吐量。
3. 改善数据局部性。运行数据放入与用户地理位置相近的地方。

Kafka的副本机制与其他分布式系统不同，Kafka的追随者副本不对外提高服务，所有的请求必须由Leader副本处理，追随者副本的唯一任务就是从Leader副本==异步拉取==消息，写入到自己的提交日志种，从而实现与Leader副本的同步。当Leader副本挂了，Kafka依托于Zookeeper提供的监控功能可以实时感知，并立即开启新一轮的Leader选举。

这种机制有两个好处：

1. 实现“Read-your-writes”，生产者写入消息后，消费者马上可以读取到，而不用等待异步同步完成。
2. 实现单调读（Monotonic Reads），由于副本的同步进度不一样，避免从不同的副本上读取到不一致的数据。

**ISR （In-Sync Replicas）**

ISR副本集合是一个动态调整的集合，集合中的副本都被认为是和Leader副本同步的，Leader副本是天然就在ISR中的（如果Leader副本挂了，集合就是空的）。Broker端参数`replica.lag.time.max.ms`含义是追随者副本能够落后Leader副本的最大时间间隔（默认10s），如果追随者副本落后超过这个时间，就会被踢出ISR集合，等重新追上了会被重新加回ISR。

所有不在ISR集合中的存活副本称为==非同步副本==，如果允许选举这种副本的过程称为==Unclean 领导者选举==，Broker端参数`unclean.leader.election.enable`可以控制是否允许非同步副本参与Leader选举。如果开启，可能回造成数据丢失，反之，则能保证数据的一致性。通过这个参数，Kafka赋予你选择CP还是AP的权利。