# 一、背景知识

## Kafka定义

传统定义：Kafka 是一个**分布式**的基于**发布/订阅**模式的**消息队列**，主要应用于大数据实时处理领域。

最新定义：Kafka 是一个开源的**分布式事件流平台**，被数千家公司用于高性能数据管道、流分析、数据集成和关键任务应用。

## 消息队列

传统的消息队列的主要应用场景包括： **缓存/消峰、 解耦和异步通信**。目前企业中比较常见的消息队列产品主要有 ActiveMQ、RabbitMQ、RocketMQ、Kafka 等。

消息队列的两种模式：

- 点对点模式：一对一，消费者主动拉取数据，消息收到后消息清除。该模式使用较少
- 发布/ 订阅模式：一对多，消息生产者将消息发布到 topic 中，同时有多个消费者消费该消息，消费之后不会清除消息

# 二、Kafka架构

![img](https:////upload-images.jianshu.io/upload_images/2599320-155b87302426f6a0.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

Kafka架构

1. Producer：消息生产者，就是向 kafka broker 发消息的客户端
2. Consumer：消息消费者，向 kafka broker 取消息的客户端
3. Consumer Group：消费者组，由多个 consumer 组成。**消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费**；消费者组之间互不影响，所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者
4. Broker：一台 kafka 服务器就是一个 broker，一个集群由多个 broker 组成。一个 broker可以容纳多个 topic
5. Topic：可以理解为一个队列，**生产者和消费者面向的都是一个 topic**
6. Partition：为了实现扩展性，一个非常大的 topic 可以分布到多个 broker（即服务器）上，**一个 topic 可以分为多个 partition，每个 partition 是一个有序的队列**
7. Replica：副本，为保证集群中的某个节点发生故障时，该节点上的 partition 数据不丢失，且 kafka 仍然能够继续工作，kafka 提供了副本机制，一个 topic 的每个分区都有若干个副本，其中有一个 leader 和若干个 follower
8. Leader：每个分区多个副本的主，生产者发送数据的对象，以及消费者消费数据的对象都是 leader。由 zk 记录谁是 leader，2.8.0 版本以后也可以配置不使用 zk
9. Follower：每个分区多个副本中的从，实时从 leader 中同步数据，保持和 leader 数据的同步。leader 发生故障时，某个 follower 会成为新的 follower。

# 三、生产者

## 3.1 消息发送流程

在消息发送的过程中，涉及到了两个线程：main 线程和 sender 线程。在 main 线程中创建了一个双端队列 RecordAccumulator。Main 线程将消息发送给 RecordAccumulator，sender 线程不断从 RecordAccumulator 中拉取消息发送到 broker。

![img](https:////upload-images.jianshu.io/upload_images/2599320-3f7ac541614d8aa1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

消息发送流程

几个重要参数：

- buffer.memory：RecordAccumulator 缓冲区总大小，默认 32m
- batch.size：缓冲区一批数据最大值，默认16k。适当增加该值，可以提高吞吐量，但是如果该值设置太大，会导致数据传输延迟增加
- linger.ms：如果数据迟迟未达到 batch.size， sender 等待 linger.time 之后就会发送数据。单位 ms，默认值是 0ms, 表示没
   有延迟。生产环境建议该值大小为 5-100ms 之间
- acks：Kafka 提供了三种可靠性级别，用户根据对可靠性和延迟的要求进行权衡，选择以下的配置：
   0：生产者发送过来的数据，不需要等数据落盘应答
   1：生产者发送过来的数据，leader 收到数据后应答
   -1（all）：生产者发送过来的数据，leader 和 ISR（和 leader 保持同步的 follower 集合） 队列里面的所有节点收齐数据后应答。 默认值是-1，-1 和 all 是等价的
- compression.type：生产者发送的所有数据的压缩方式。默认是 none，也就是不压缩。支持压缩类型：none、gzip、snappy、lz4 和 zstd
- max.in.flight.requests.per.connection：允许最多没有返回 ack 的次数，默认为 5，开启幂等性要保证该值是 1-5 的数字

几种消息发送方式：

- 普通异步发送
- 带回调函数的异步 api
- 同步 api

## 3.2 分区

分区的好处：

- 方便在集群中**扩展**，每个 partition 可以通过调整以适应它所在的机器，而一个 topic 又可以有多个 Partition 组成，因此整个集群就可以适应任意大小的数据了
- 可以**提高并发**，因为可以以 partition 为单位生产/消费数据了

生产者发送消息的分区策略：

1. 指明 partition 的情况下，直接将指明的值直接作为 partiton 值
2. 没有指明 partition 值但有 key 的情况下，将 key 的 hash 值与 topic 的 partition 数进行取余得到 partition 值
3. 既没有 partition 值又没有 key 值的情况下，第一次调用时随机生成一个整数（后面每次调用在这个整数上自增），将这个值与 topic 可用的 partition 总数取余得到 partition 值，也就是常说的 round-robin 轮询算法

## 3.3 生产经验

### 生产者如何提高吞吐量

1. 调整批次大小：如将 batch.size 由16k调整为32k
2. 调整Sender线程等待时间：如将 linger.ms 由0调整为5-100ms
3. 压缩策略：如将 compression.type 设为 snappy
4. 调整缓存大小：如将 buffer.memory 由32m调整为64m

### 数据可靠性

Ack应答级别：

- acks=0，生产者发送数据后就不管了，可靠性差，效率高
- acks=1，生产者发送数据后 leader 应答即可，可靠性中等，效率中等
- acks=-1，生产者发送数据后 leader 和 ISR 队列中所有 follower 应答才行，可靠性高，效率低

生产环境中，acks=0 很少使用；acks=1，一般用于传输普通日志，允许丢失个别数据；acks=-1，一般用于传输和交易相关等对可靠性要求较高的场景。

数据完全可靠条件 = ACK级别为-1 + 分区副本大于等于2 + ISR里应答的最小副本数大于等于2

### 数据重复性

至少一次（At Least Once）= ACK级别为1 + 分区副本大于等于2 + ISR里应答的最小副本数大于等于2。不能保证数据不重复。

最多一次（At Most Once）= ACK级别为0。不能保证数据不丢失。

精确一次（Exactly Once）= 幂等性 + 至少一次。幂等性默认开启，但只能保证在单分区单会话内不重复，如果需要全局严格一致，则需要开启事务（开启事务的前提是开启幂等性）。

### 数据顺序

单分区内，可以配置为有序：多分区，分区与分区间无序。

单分区有序的条件：

- 1.x 版本之前：max.in.flight.requests.per.connection = 1
- 1.x 及之后版本：
   （1）若未开启幂等性
   配置 max.in.flight.requests.per.connection = 1
   （2）若开启幂等性
   配置 max.in.flight.requests.per.connection <= 5。其原理是 1.x 版本后，如果开启幂等，kafka 服务端会缓存生产者发来的最近5个 requests 的元数据，因此可以保证最近5个 requests 的数据是有序的。

# 四、Broker

## 4.1 Broker启动流程

Kafka 集群中有一个 broker 的 controller 会被选举为 controller leader，负责管理集群 broker 的上下线、所有 topic 的分区副本分配和 leader 选举等工作。Controller 的信息同步工作是依赖于 zookeeper 的（2.8.0 版本以后也可以不依赖）。

![img](https:////upload-images.jianshu.io/upload_images/2599320-bbe60bc399c84c4b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1087/format/webp)

Broker启动流程

## 4.2 副本与故障处理

### 副本

副本的作用是提高数据可靠性，Kafka 默认副本1个，生产环境一般配置为2个，保证数据可靠性；太多副本会增加磁盘存储空间，增加网络上数据传输，降低效率。

Kafka 中副本分为：leader 和 follower。Kafka 生产者只会把数据发往 leader，
 然后 follower 找 leader 进行同步数据。

几个重要概念：

- AR：Kafka 分区中的所有副本统称为（Assigned Repllicas）。AR = ISR + OSR
- ISR：表示和 leader 保持同步的 follower集合。如果 follower 长时间未向 leader 发送通信请求或同步数据，则该 follower 将被踢出 ISR。该时间阈值由 replica.lag.time.max.ms 参数设定，默认30s。Leader 发生故障之后，就会从 ISR 中选举新的 leader
- OSR：表示 follower 与 leader 副本同步时，延迟过多的副本
- LEO：Log End Offset，每个副本的最新的 offset + 1
- HW：High Watermart，所有副本中最小的 LEO

### Follower 故障

1. Follower 发生故障后会被临时提出 ISR
2. 这个期间 leader 和 follower 继续接受数据
3. 待该 follower 恢复后，follower 会读取本地磁盘记录的上次的 HW，并将 log 文件高于 HW 的部分截取掉，从 HW 开始向 leader 进行同步
4. 等该 follower 的 LEO 大于等于该分区的 HW，即 follower 追上 leader 之后，就可以重新加入 ISR 了

### Leader 故障

1. Leader 发生故障之后，会从 ISR 中选出一个新的 leader
2. 为保证多个副本之间的数据一致性，其余的 follower 会先将各自的 log 文件高于 HW 的部分截掉，然后从新的 leader 同步数据

注意： 这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复。如何保证？见上一节数据可靠性。

## 4.3 文件存储

Topic 是逻辑上的概念，而 partition 是物理上的概念，每个 partition 对应一个 log 文件，该文件中存储的就是 producer 生产的数据。Producer 生产的数据会不断**追加到该 log 文件末端**。为防止 log 文件过大导致数据定位效率低下，kafka 采取了**分片和索引**机制，将每个 partition 分为多个 segment。每个 segment 包括：.index 文件、.log 文件和 .timeindex 等文件，这些文件位于一个文件夹下，该文件夹命名规则：topic 名称 + 分区序号，例如：first-0。

![img](https:////upload-images.jianshu.io/upload_images/2599320-33df5b740dae009b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1087/format/webp)

文件存储机制

两个重要参数：

- log.segment.bytes：log 日志划分成块（即 segment）的大小，默认值1G
- log.index.interval.bytes：默认4kb，每当写入了4kb大小的日志（.log），然后就往 index 文件里面记录一个索引（稀疏索引）

### Log 文件和 Index 文件示例

![img](https:////upload-images.jianshu.io/upload_images/2599320-4c6aa1efcee68c18.png?imageMogr2/auto-orient/strip|imageView2/2/w/737/format/webp)

文件示例

### 高效读写数据

Kafka 如何做到高效读写数据？

1. Kafka 本身是分布式集群，可以采用分区技术，并行度高
2. 读数据采用稀疏索引，可以快速定位要消费的数据
3. 顺序写磁盘，生产者数据是一直追加到 log 文件末端的顺序写（顺序写 600M/s vs 随机写 100K/s）
4. 零拷贝+页缓存技术
    零拷贝：Kafka 的数据加工处理由生产者和消费者处理，broker 应用层不关心存储的数据，所以就不用了走应用层，传输效率高。
    页缓存：操作系统提供，当上层由写操作时，操作系统只是将数据写入 PageCache；读操作时先从 PageCache 中查找，找不到再去磁盘中获取。

关于零拷贝和页缓存，具体可以参考：[https://zhuanlan.zhihu.com/p/258513662](https://links.jianshu.com/go?to=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F258513662)

# 五、消费者

## 5.1 消费方式

Consumer 采用 pull（拉）模式从 broker 中读取数据；因为 push （推）模式很难适应消费速率不同的消费者。

Pull 模式不足之处是，如果 kafka 没有数据，消费者可能会陷入循环中，一直返回空数据。针对这一点，kafka 的消费者在消费数据时会传入一个时长参数 timeout，如果当前没有数据可供消费，consumer 会等待一段时间之后再返回，这段时长即为 timeout。

## 5.2 消费者组

消费者组（Consumer Group，CG）由多个 consumer 组成。形成一个消费者组的条件，是所有消费者的 groupid 相同。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响，所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。

消费者组初始化流程：



![img](https:////upload-images.jianshu.io/upload_images/2599320-95a1ad65d2a2174e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1082/format/webp)

消费者组初始化流程

消费者组消费流程：



![img](https:////upload-images.jianshu.io/upload_images/2599320-c84c73484092c7b8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1093/format/webp)

消费者组消费流程

## 5.3 分区的分配与再平衡

一个消费者组中有多个 consumer，一个 topic 有多个 partition，所以必然会涉及到 partition 的分配问题，即确定那个 partition 由哪个 consumer 来消费。当消费者组里面的消费者个数发生改变的时候，也会触发再平衡。

Kafka 有四种分配策略，可以通过参数 partition.assignment.strategy 来配置，默认 Range + CooperativeSticky。

- Range：针对每个 topic。将 topic 中的分区与消费者排序，通过分区数/消费者数决定每个消费者消费几个分区，若除不尽则前面几个消费者会多消费1个分区。注意，如果有N个 topic，容易产生数据倾斜
- RoundRobin：针对集群中的所有 topic。把所有分区和所有的消费者都列出来，然后按照 hashcode 进行排序，最后通过轮训算法来分配分区给到各个消费者
- Sticky：粘性分区从 0.11.x 版本开始引入，首先会尽量均衡的放置分区到消费者上面，在出现同一消费者组内消费者出现问题的时候，会尽量保持原有分配的分
   区不变化
- CooperativeSticky：和 sticky 类似只是支持了cooperative 的 再平衡

## 5.4 Offset

由于 consumer 在消费过程中可能会出现断电宕机等故障，consumer 恢复后，需要从故障前的位置的继续消费，所以 consumer 需要实时记录自己消费到了哪个 offset，以便故障恢复后继续消费。

Kafka 0.9版本之前，consumer 默认将 offset 保存在 zookeeper 中；从 0.9 版本开始，默认将 offset 保存在 kafka 一个内置的 topic 中，该 topic 为__consumer_offsets。__consumer_offsets 主题里面采用 key 和 value 的方式存储数据。Key 是 group.id+topic+分区号，value 就是当前 offset 的值。 每隔一段时间，kafka 内部会对这个 topic 进行 compact，也就是每个 group.id+topic+分区号 就保留最新数据。

### 提交 offset

- 自动提交：为了使用户专注自己的业务逻辑，kafka 提供了自动提交 offset 的功能，相关参数：
   enable.auto.commit：是否开启自动提交，默认 true
   auto.commit.inteval.ms：自动提交的时间间隔，默认5s
- 手动提交：包括两种方式，同步提交（commitSync）和异步提交（commitAsync）

重复消费： 已经消费了数据，但是 offset 没提交。
 漏消费： 先提交 offset 后消费，有可能会造成数据的漏消费。

如何避免漏消费和重复消费，做到精准一次消费呢？这依赖于消费者事务，要求消费端将消费过程和提交 offset 过程做原子绑定，也就是说需要将 offset 保存到支持事务的自定义介质（如 Mysql）。

### 指定 offset 消费

当 kafka 中没有初始偏移量（消费者组第一次消费）或服务器上不再存在当前偏移量时（例如该数据已被删除），该怎么办？有以下几种配置：

- earliest：自动将偏移量重置为最早的偏移量
- latest（默认值）：自动将偏移量重置为最新偏移量
- none：如果未找到消费者组的先前偏移量，则向消费者抛出异常
- 任意指定 offset 位移开始消费

## 5.5 生产经验

### 如何提高吞吐量（避免数据积压）

- 如果是消费能力不足，可以考虑增加 topic 的分区数，并提升消费者组的消费者数量，使消费者数 = 分区数
- 如果是下游的数据处理不及时，可以提高每批次拉取的数量。如果拉取数据/处理时间 < 生产速度，即处理的数据小于生产的数据，也会造成数据积压

# 六、Kafka-Kraft 模式

![img](https:////upload-images.jianshu.io/upload_images/2599320-e1a1495520d9e1e5.png?imageMogr2/auto-orient/strip|imageView2/2/w/509/format/webp)

kafka架构

左图为 kafka 原有架构，元数据在 zookeeper 中，运行时动态选举 controller，由 controller 进行 kafka 集群管理。右图为 kraft 模式架构（实验性），不再依赖 zookeeper 集群，而是用三台 controller 节点代替 zookeeper，元数据保存在 controller 中，由 controller 直接进行 kafka 集群管理。这样做的好处有以下几个：

- Kafka 不再依赖外部框架，而是能够独立运行
- Controller 管理集群时，不再需要从 zookeeper 中先读取数据，集群性能上升
- 由于不依赖 zookeeper，集群扩展时不再受到 zookeeper 读写能力限制
- Controller 不再动态选举，而是由配置文件规定。这样我们可以有针对性的加强
   controller 节点的配置，而不是像以前一样对随机 controller 节点的高负载束手无策