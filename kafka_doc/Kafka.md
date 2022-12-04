# 基础架构

Kafka的数据单元称为消息。可以把消息看成是数据库里的一个“数据行”或一条“记录”，消息由字节数组组成。

消息有键，键也是一个字节数组。当消息以一种可控的方式写入不同的分区时，会用到键。

**批量写入**

为了提高效率，消息被分批写入Kafka。批次就是一组消息，这些消息属于同一个主题和分区，把消息分成批次可以减少网络开销。

1， 批次越大，单位时间内处理的消息就越多，单个消息的传输时间就越长。

2， 批次数据会被压缩，这样可以提升数据的传输和存储能力，但是需要更多的计算处理。

**主题和分区**

Kafka的消息通过主题进行**分类，可比是数据库的表或者文件系统里的文件夹。

分区：主题可以被分为若干分区，一个主题通过分区分布于Kafka集群中，提供了横向扩展的能力。

![image-20221106173113733](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221106173113733.png)

**生产者和消费者**

一个消息被生产者生产，必须得发布到一个特定的主题上。

默认把生产者的消息均衡地分布在主题的所有分区，也可有选择不同的写入分区的方式

1. 直接指定消息的分区（自定义分区器）
2. 根据消息的key散列取模得出分区（通过分区器实现）
3. 轮询指定分区。

*消费者和消费者组*

通过**偏移量**来区分已经读过的消息，从而消费消息。

1. 消费者订阅一个或多个主题，并按照消息生成的顺序读取它们。
2. 消费者通过检查消息的**偏移量**来区分已经读取过的消息。
3. 偏移量是另一种元数据，它是一个**不断递增的整数值**，在创建消息时，Kafka 会把它添加到消息里。
4. 在给定的**分区**里，每个消息的 偏移量都是**唯一**的。消费者把每个分区最后读取的消息偏移量保存在Zookeeper 或**Kafka** 上，如果消费者关闭或重启，它的读取状态不会丢失。

消费者组包含多个消费者，消费组保证每个分区只能被一个消费者使用（一个消费者也可以消费多个分区），避免重复消费，如果一个消费者失效，消费组里的其他消费者可以接管失效消费者的工作，再平衡，分区重新分配（重平衡）

![image-20221106173542363](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221106173542363.png)

**broker**

一个独立的Kafka服务器称为broker，broker接收来自生产者的消息，**为消息设置偏移量**，并提交消息到磁盘保存（顺序保存）。

为消费者提供服务，对读取分区的请求做出响应，**返回已经提交到磁盘上的消息。**

单机性能：**单个**broker**可以轻松处理**数千个分区**以及**每秒百万级的消息量。

1， 如果某topic有N个partition，集群有N个broker，那么每个broker存储该topic的一个 partition。

2， 如果某topic有N个partition，集群有(N+M)个broker，那么其中有N个broker存储该topic的一 个partition，剩下的M个broker不存储该topic的partition数据。

3， 如果某topic有N个partition，集群中broker数目少于N个，那么一个broker存储该topic的一 个或多个partition。在实际生产环境中，尽量避免这种情况的发生，这种情况容易导致Kafka 集群数据不均衡。

![image-20221106174309618](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221106174309618.png)

集群中一个分区属于一个**broker**，该broker称为**分区首领**。

**一个分区**可以分配给**多个**broker，此时会发生分区复制。

分区的复制提供了**消息冗余，高可用**。

>  **副本分区**不负责处理消息的读写。

**topic**

1，每条发布到Kafka集群的消息都有一个类别，这个类别被称为Topic。

2， 物理上不同Topic的消息分开存储。

**partition**

1. 主题可以被分为若干个分区，一个分区就是一个提交日志。
2. 消息以追加的方式写入分区，然后以先入先出的顺序读取（顺序性）
3. 无法在整个主题范围内保证消息的顺序，但可以保证消息在单个分区内的顺序。（单个分区的消息有序）
4. Kafka 通过分区来实现数据冗余和伸缩性。（高可用）
5.  在需要严格保证消息的消费顺序的场景下，需要将partition数目设为1。

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/image-20221106181840749.png" alt="image-20221106181840749" style="zoom:50%;" />

**副本（Replicas）**

每个主题被分为若干个分区，每个分区有多个副本，副本保存在brooder上

每个broker可以保存数百个属于不同主题和不同分区的副本（同一个主题的分区A，分区A的副本不可以和A在同一台broker上，这样就失去了分区的意义）

*类型*

首领副本：每个分区都有一个首领副本，为了保证一致性，所有**生产者请求和消费者请求**都会经过这个副本。

跟随者副本：跟随者副本不处理来自客户端的请求，它们唯一的任务就是从首领那里复制消息，保持与首领一致的状态。如果首领发生崩溃，其中的一个跟随者会被提升为新首领。

跟随者副本包括**同步副本**和**不同步副本**，在发生首领副本切换的时候，只有同步副本可以切换为首领副本。

**AR**

- 分区中的所有副本统称为**AR**(Assigned Repllicas)。**AR=ISR+OSR**

**ISR**

- 所有与leader副本保持**一定程度**同步的副本(包括Leader)组成**ISR**(In-Sync Replicas)，ISR集合 是AR集合中的一个子集。
- 消息会先发送到leader副本，然后follower副本才能从leader副本中拉取消息 进行同步，同步期间内follower副本相对于leader副本而言会有一定程度的滞后。
- 所说的“一定程度” 是指可以忍受的滞后范围，这个范围可以通过参数进行配置。

**OSR**

- 与leader副本同步滞后过多的副本(不包括leader)副本，组成**OSR**(Out-Sync Relipcas)。
- 在正常情况下，所有的follower副本都应该与leader副本保持一定程度的同步，即AR=ISR,OSR集合为空。

**HW**

-  俗称高水位，它表示了一个特定消息的偏移量(offset)，**消费者只能拉取到这个offset之前的消息。**

 **LEO**

-  LEO是Log End Offset的缩写，它表示了当前日志文件中**下一条待写入**消息的offset。

​	<img src="https://gitee.com/JieMingLi/document-pics/raw/master/image-20221106183805791.png" alt="image-20221106183805791" style="zoom:50%;" />

**offset（偏移量）**

生产者offset

消息写入的时候，每一个分区都有一个offset，这个offset就是生产者的offset，同时也是这个分区 的最新最大的offset。

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/image-20221106182940145.png" alt="image-20221106182940145" style="zoom: 67%;" />

消费者offset

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/image-20221106183111291.png" alt="image-20221106183111291" style="zoom:67%;" />

生产者写入的offset是最新最大的值是12

1， Consumer A进行消费时，从0开始消费，一直消费到了9，消费者的offset就记录在9。

2， Consumer B就纪录在了11。

3， 等下一次他们再来消费时，他们可以选择接着上一次的位置消费，当然也可以选择从头消费，或者跳到最近的 记录并从“现在”开始消费。



# 消费者

自动提交

手动递交

- 同步提交
- 异步提交





------

消费者自动提交带来的问题

消费者协调者的作用

协调者：Coordinator，它专门为 Consumer Group 服务，负责为 Group 执行 Rebalance 以及提供位移管理和组成员管理等。

Consumer 端应用程序在提交位移时，其实是向 Coordinator 所在的 Broker 提交位移。当 Consumer 应用启动时，也是向 Coordinator 所在的 Broker 发送各种请求，然后由 Coordinator 负责执行消费者组的注册、成员管理记录等元数据管理操作。

所有 Broker 在启动时，都会创建和开启相应的 Coordinator 组件。也就是说，**所有 Broker 都有各自的 Coordinator 组件**

确认协调者：

1， 确定由位移主题的哪个分区来保存该 Group 数据：partitionId=Math.abs(groupId.hashCode() % offsetsTopicPartitionCount)。

2， 找出该分区 Leader 副本所在的 Broker，该 Broker 即为对应的 Coordinator。



# 消费者组

consumer group是kafka提供的可扩展且具有容错性的消费者机制。

**三个特性:**

1. 消费组有一个或多个消费者，消费者可以是一个进程，也可以是一个线程 
2.  group.id是一个字符串，唯一标识一个消费组
3.  消费组订阅的主题每个分区只能分配给消费组一个消费者

**位移**

消费者在消费的过程中记录已消费的数据，即消费位移(offset)信息。每个**消费组保存自己的位移信息**，只需要简单的一个整数表示位置就够了，同时可以引入 checkpoint机制定期持久化。

>  位移保存在kafka的位移数据结构：K-V形势，key：grouId + topic + partition， value：这个分区消费的偏移量

**位移管理**(offset management)

Kafka默认定期自动提交位移( enable.auto.commit = true )，也手动提交位移。另外kafka会定期把group消费情况保存起来，做成一个offset map

![image-20221201165951604](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221201165951604.png)

**位移提交**

位移是提交到Kafka中的 __consumer_offsets 主题。 __consumer_offsets 中的消息保存了每个消费组某一时刻提交的offset信息。

> 为什么位移不提交到zk？ZooKeeper 其实并不适用于这种高频的写操作，为什么zk不适合高频写操作？

![image-20221201170430034](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221201170430034.png)

表示消费组为 test-consumer-group ，消费的主题为 __consumer_offsets ， 消费的分区是4，偏移量为5。

__consumers_offsets 主题配置了compact策略，（[存在同一个key，多个不同的值的问题，但是这个问题如何触发](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Kafka%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%AE%9E%E6%88%98/16%20%20%E6%8F%AD%E5%BC%80%E7%A5%9E%E7%A7%98%E7%9A%84%E2%80%9C%E4%BD%8D%E7%A7%BB%E4%B8%BB%E9%A2%98%E2%80%9D%E9%9D%A2%E7%BA%B1.md)）使得它总是能够保存最新的位移信息，既控制 了该topic总体的日志容量，也能实现保存最新offset的目的。

## [再均衡](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Kafka%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%AE%9E%E6%88%98/25%20%20%E6%B6%88%E8%B4%B9%E8%80%85%E7%BB%84%E9%87%8D%E5%B9%B3%E8%A1%A1%E5%85%A8%E6%B5%81%E7%A8%8B%E8%A7%A3%E6%9E%90.md)

再均衡(Rebalance)本质上是一种协议，规定了一个消费组中所有消费者如何达成一致来分配订 阅主题的每个分区。

> 比如某个消费组有20个消费组，订阅了一个具有100个分区的主题。正常情况下，Kafka平均会为每 个消费者分配5个分区。这个分配的过程就叫再均衡。

**发生条件**

1. **组成员**发生变更(新消费者加入消费组组、已有消费者主动离开或崩溃了)
2. 订阅**主题数**发生变更。如果正则表达式进行订阅，则新建匹配正则表达式的主题触发再均衡。 
3. 订阅主题的**分区数**发生变更

**分配的策略**

RangeAssignor和RoundRobinAssignor以及StickyAssignor。

**谁来执行再均衡**

Kafka提供了一个角色:Group Coordinator来执行对于消费组的管理。

Group Coordinator：每个消费组分配一个消费组协调器用于**组管理和位移管理**。当消费组的第1个消费者启动的时候，它会去和Kafka Broker确定谁是它们组的组协调器。之后该消费组内所有消费者和该组协调器协调通信。

这个组协调器就是指borker上的一台机器。

计算方式

- 确定消费组位移信息写入 __consumers_offsets 的哪个分区。
  - __consumers_offsets partition# = Math.abs(groupId.hashCode() % groupMetadataTopicPartitionCount) 注意:groupMetadataTopicPartitionCount 由 offsets.topic.num.partitions 指定，默认是50个分区。
- 该分区leader所在的broker就是组协调器。

> [组协调器](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Kafka%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%AE%9E%E6%88%98/17%20%20%E6%B6%88%E8%B4%B9%E8%80%85%E7%BB%84%E9%87%8D%E5%B9%B3%E8%A1%A1%E8%83%BD%E9%81%BF%E5%85%8D%E5%90%97%EF%BC%9F.md)

**如何再均衡**

*Rebalance Generation*

它表示Rebalance之后主题分区到消费组中消费者映射关系的一个版本，主要是用于保护消费组， 隔离无效偏移量提交的。如上一个版本的消费者无法提交位移到新版本的消费组中，因为映射关系变 了，消费者消费的或许已经不是原来的那个分区了。每次group进行Rebalance之后，Generation号都会加 1，表示消费组和分区的映射关系到了一个新版本。

*协议*

kafka提供了5个协议来处理与消费组协调相关的问题:

1. Heartbeat请求:consumer需要定期给组协调器发送心跳来表明自己还活着 
2. LeaveGroup请求:主动告诉组协调器我要离开消费组 
3. SyncGroup请求:消费组Leader把分配方案告诉组内所有成员 
4. JoinGroup请求:成员请求加入组 
5. DescribeGroup请求:显示组的所有信息，包括成员信息，协议名称，分配方案，订阅信息 等。通常该请求是给管理员使用

消费者通过定时向消费组协调器发送Heartbeat请求。如果超过了设定的超时时间，那么协调器认为该消费者已经挂了。一旦协调器认为某个消费者挂了（就是满足再平衡的条件），那么它就会开启新一轮再均衡，并且在当前其他消费者的心跳**响应中**添加“REBALANCE_IN_PROGRESS”，告诉 其他消费者:重新分配分区。

*状态机转换*

Empty：组内没有任何成员，但消费者组可能存在已提交的位移数据，而且这些位移尚未过期。

Dead：同样是组内没有任何成员，但组的元数据信息已经在**协调者**端被移除。协调者组件保存着当前向它注册过的所有组信息，所谓的元数据信息就类似于这个注册信息。

PreparingRebalance:组准备开启新的rebalance，等待成员加入

CompletingRebalance :消费者组下所有成员已经加入，各个成员正在等待分配方案。该状态在老一点的版本中被称为AwaitingSync，它和Completing Rebalance是等价的。

Stable:再均衡完成，可以开始消费。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/f16fbcb798a53c21c3bf1bcd5b72b006.png)

注意：

1. 当有新成员加入或已有成员退出时，消费者组的状态从 Stable 直接跳到 PreparingRebalance 状态，**所有现存成员就必须重新申请加入组**
2. 当所有成员都退出组后，消费者组状态变更为 Empty。Kafka 定期自动删除过期位移的条件就是，组要处于 Empty 状态。

*过程*

1. Join加入组。所有成员都向消费组协调器发送JoinGroup请求，请求加入消费组。所有成员都发送了JoinGroup请求，协调i器从中选择一个消费者担任Leader的角色，并把组成员信息以及订阅信息发给Leader。

   ![image-20221201211744560](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221201211744560.png)

   

2. Sync，Leader开始分配消费方案，即哪个消费者负责消费哪些主题的哪些分区。一旦完成分配，Leader会将这个方案封装进SyncGroup请求中发给消费组协调器，非Leader也会发 SyncGroup请求，只是内容为空。**消费组协调器接收到分配方案之后会把方案塞进SyncGroup 的response中发给各个消费者。**

   ![image-20221201211902655](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221201211902655.png)

> 消费组的分区分配方案在客户端执行。Kafka交给客户端可以有更好的灵活性。Kafka默认提 供三种分配策略:range和round-robin和sticky。可以通过消费者的参数:partition.assignment.strategy 来实现自己分配策略。

# 分区

## 副本

Kafka在一定数量的服务器上对主题分区进行复制。当集群中的一个broker宕机后系统可以自动故障转移到其他可用的副本上，不会造成数据丢失。

![image-20221203105546283](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221203105546283.png)

>  Follower分区像 **Kafka** 一样消费来自Leader分区的消息，并将其持久化到自己的日志。允许Follower对日志条目拉取进行**批处理**。

### 特性

1. 主题的分区是复制的最小单元。
2.  在非故障情况下，Kafka中的每个分区都有一个Leader副本和零个或多个Follower副本。
3. 复制因子：Leader+其他副本
4. 所有读取和写入都由Leader副本负责，副本分区只负责同步leader分区的数据，不做任何的读写。
5. 分区比broker多，并且Leader分区在broker之间平均分配（了解一下分配的算法）。

**同步副本（ISR）**

1. 节点必须能够维持与ZooKeeper的会话(通过ZooKeeper的心跳机制)

2. 对于Follower副本分区，它复制在Leader分区上的写入，并且不要延迟太多（也会有丢失数据的风险，比如刚写到leader，ISR的数据还没开始同步就丢了，[参考](https://codeantenna.com/a/CXYAW6vBNa)）
3. Kafka提供的保证是，只要有至少一个同步副本处于活动状态，提交的消息就不会丢失。（不像zk需要过半的机制）

**Leader宕机后如何恢复**

1   当leader宕机了，会从follower选择一个作为leader。当宕机的重新恢复时，会把之前commit的数据清空，重新从新的leader里pull数据。

2 全部副本宕机，有2种恢复方式。

（一）等待ISR中的一个恢复后，并选它作为leader。(*等待时间较长，降低可用性*)

（二）选择第一个恢复的副本作为新的leader，无论是否在ISR中。(*并未包含之前leader commit的 数据，因此造成数据丢失*)

### Leader选举

分区P1的Leader是0，ISR是0和1 

分区P2的Leader是2，ISR是1和2 

分区P3的Leader是1，ISR是0，1，2。

**ISR包含着Leader。**

![image-20221203115831878](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221203115831878.png)

如果某个分区所在的服务器除了问题，不可用，kafka会从该分区的其他的副本中选择一个作为新的 Leader，之后所有的读写就会转移到这个新的Leader上。

>  选择哪一个分区副本作为新的Leader？

如果某个分区的Leader不可用，Kafka就会从ISR集合中选择一个副本作为新的Leader。

Kafka会在Zookeeper上针对每个Topic维护一个称为ISR(in-sync replica，已同步的副本)的集合，该集合中是一些分区的副本，如果这个集合有增减，kafka会通过zk同步到消息并更新记录。

通过ISR，kafka需要的**冗余度较低**，可以容忍的失败数比较高，假设某个topic有N+1个副本，kafka可以容忍N个服务器不可用（不像zk需要过半的机制）

> 为什么kafka的容忍性不用过半机制

少数服从多数是一种比较常见的一致性算发和Leader选举法。 它的含义是只有超过半数的副本同步了，系统才会认为数据已同步; 选择Leader时也是从超过半数的同步的副本中选择。 **这种算法需要较高的冗余度，跟Kafka比起来，浪费资源**。 譬如只允许一台机器失败，需要有三个副本;而如果只容忍两台机器失败，则需要五个副本。 而kafka的ISR集合方法，分别只需要两个和三个副本。

## 分配的策略

每个Topic会包含多个分区，默认情况下一个分区只能被一个消费组下面的一个消费者消费，这里就产生了分区分配的问题。Kafka中提供了多重分区分配算法(PartitionAssignor)的实现: RangeAssignor、RoundRobinAssignor、StickyAssignor。

**RangeAssignor**

*Kafka**默认采用RangeAssignor的分配算法**。针对单个topic。

分配的规则

- RangeAssignor对每个Topic进行独立的分区分配。

- 对于每一个Topic，首先对分区按照分区ID进行 数值排序，然后订阅这个Topic的消费组的消费者再进行字典排序，之后尽量均衡的将分区分配给消费 者。

原理：按照消费者总数和分区总数进行整除运算来获得一个跨度，然后将分 区按照跨度进行平均分配，以保证分区尽可能均匀地分配给所有的消费者。对于每一个Topic， RangeAssignor策略会将消费组内所有订阅这个Topic的消费者按照名称的字典序排序，然后为每个消费 者划分固定的分区范围，如果不够平均分配，那么字典序靠前的消费者会被多分配一个分区。

![image-20221204203907670](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221204203907670.png)

优点：尽量均衡的将分区分配给消费 者。这里只能是尽量均衡，因为分区数可能无法被消费者数量整除

缺点：随着消费者订阅的Topic的数量的增加，不均衡的问题会越来越严重，比如图中4个分区3个消费者的场景，C0会多分配一个分区。如果此时再订阅一个分区数为4的 Topic，那么C0又会比C1、C2多分配一个分区，这样C0总共就比C1、C2多分配两个分区了，而且随着 Topic的增加，这个情况会越来越严重。

> 字典序靠前的消费组中的消费者比较“**贪婪**”。

![image-20221204204115472](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221204204115472.png)

**RoundRobinAssignor（针对多个topic）**

1. 将消费组内订阅的**所有Topic的分区**及所有消费者进行排序后尽 量均衡的分配(RangeAssignor是针对单个Topic的分区进行排序分配的)
2. 如果消费组内，消费者订阅 的Topic列表是相同的(每个消费者都订阅了相同的Topic)，那么分配结果是尽量均衡的(消费者之间 分配到的分区数的差值不会超过1)。
3. 如果订阅的Topic列表是不同的，那么分配结果是不保证“尽量均 衡”的，因为某些消费者不参与一些Topic的分配。

![image-20221204204340107](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221204204340107.png)

**优点**：在订阅多个Topic的情况下，RoundRobinAssignor的方式能消费者之间**尽量均衡的分配到分区**(分配到的分区数的差值不会超过1——RangeAssignor的分配策略可能随着订阅的 Topic越来越多，差值越来越大)。

**缺点**：对于消费组内消费者订阅Topic不一致的情况，在重新分配的时候，没有考虑重新分配前现状。

举例：假设有两个个消费者分别为C0和C1，有2个Topic T1、T2，分别拥有3和2个分区，并且C0订阅T1和T2，C1订阅T2，那么RoundRobinAssignor的分配结果

![image-20221204204715616](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221204204715616.png)

看上去分配已经尽量的保证均衡了，不过可以发现C0承担了4个分区的消费而C1订阅了T2一个分区，把T2P0交给C1消费能更加的均衡。



**StickyAssignor**

无论是RangeAssignor，还是RoundRobinAssignor，当前的分区分配算法都没有 考虑**上一次的分配结果**。显然，在执行一次新的分配之前，如果能考虑到上一次分配的结果，尽量少的 调整分区分配的变动，显然是能节省很多开销的。

*<u>目标</u>*

1. **分区的分配尽量的均衡**
2. **每一次重分配的结果尽量与上一次分配结果保持一致**

> 当这两个目标发生冲突时，优先保证第一个目标。第一个目标是每个分配算法都尽量尝试去完成的，而第二个目标才真正体现出StickyAssignor特性的。

有3个Consumer:C0、C1、C2 

有4个Topic:T0、T1、T2、T3，

每个Topic有2个分区 

所有Consumer都订阅了这4个分区

![image-20221204204958788](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221204204958788.png)

如果消费者1宕机，则按照RoundRobin的方式分配结果打乱从新来过，轮询分配

![image-20221204205107158](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221204205107158.png)

按照Sticky的方式仅对消费者1分配的分区进行重分配，红线部分。最终达到均衡的目的。

![image-20221204205133585](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221204205133585.png)

# 物理存储

Kafka 消息是以主题为单位进行归类，各个主题之间是彼此独立的，互不影响。

每个主题又可以分为一个或多个分区，每个分区各自存在一个记录消息数据的日志文件。

## Log Segment

### 组成

1个segment就包括.index.和.log和.timeIndex文件

1. 每个 partition 对应于一个 log 文 件，该 log 文件中存储的就是 producer 生产的数据。Producer 生产的数据会被不断追加到该 log 文件末端，且每条数据都有自己的 offset
2. 进行日志删除的时候和数据查找的时候可以快速定位。
3. ActiveLogSegment 是活跃的日志分段，拥有文件拥有写入权限，其余的 LogSegment 只有只读的权限。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/905646-20220927095130714-850494120.png)

| 文件类别                | 作用                                                |
| ----------------------- | --------------------------------------------------- |
| .index                  | 消息的物理地址的偏移量索引文件                      |
| .timeindex              | 映射时间戳和相对offset                              |
| .log                    | 日志文件(消息存储文件)，**顺序写入**。              |
| .snapshot               | 对幂等型或者事务性producer所生成的快照文件          |
| leader-epoch-checkpoint | 保存了每一任leader开始写入消息时的offset,会定时更新 |

### 特征

kafka日志追加是**顺序写入**的，.index 和 .log 文件以当前 segment 的第一条消息的偏移量命名,偏移量是一个64位的长整型数值，固定是20位数字，长度未达到，用0进行填补。

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/image-20221204235848875.png" alt="image-20221204235848875" style="zoom:67%;" /> 

**日志与索引文件的配置项说明**

| 配置项                   | 作用                                                         | 默认值         |
| ------------------------ | ------------------------------------------------------------ | -------------- |
| log.index.size.max.bytes | 索引文件最大大小，触发偏移量索引文件或时间戳索引文件分段字   | 10485760(10MB) |
| log.segment.bytes        | 分段日志文件大小                                             | 默认1G         |
| log.index.interval.bytes | 索引间隔吗，增加索引项字节间隔密度，会影响索引文件中         | 默认4K         |
| log.roll.hours           | 当前日志分段中消息的最大时间戳与当前系统 的时间戳的差值允许的最大范围，单位小时 | 168(7天        |

1. **偏移量索引文件**：用于记录消息偏移量与物理地址之间的映射关系，可以通过修改 log.index.interval.bytes 的值，改变索引项的密度。

2. **时间戳索引文件**：根据时间戳查找对应的偏移量。

3. **索引文件的写入时机**：Kafka 中的索引文件是以稀疏索引的方式构造消息的索引，并不保证每一个消息在索引文件中都有对应的索引项，因为每当写入一定量4K(log.index.interval.bytes设定)的消息时，偏移量索引文件和时间戳索引文件分别增加一个偏移量索引项和时间戳索引项。

>  *为什么索引文件以稀疏索引的方式构造？*
>
> - 因为如果把对应片段的所有消息的索引都存储，那么必然会占用大量的磁盘空间和内存。

### **切分文件的时机**

日志文件持续写入的话会空间会一直占得更多，为了方便更好地查找和压缩，需要对日志进行切人，下面几点是触发切分的场景：

1. 分段文件日志过大：当前日志**分段文件的大小**超过了 broker 端参数 log.segment.bytes 配置的值。
   - log.segment.bytes 参数的默认值为 1073741824，即 1GB。
2. 日志分段存储的时间过长：当前**日志分段中消息的最大时间戳与当前系统的时间戳的差值**大于 log.roll.ms 或log.roll.hours 参数配置的值。如果同时配置了 log.roll.ms 和 log.roll.hours 参数，那么 log.roll.ms 的优先级高。默认情况下，只配置了 log.roll.hours 参数，其值为168，即 7 天。
3. 索引文件大小过大：偏移量索引文件或时间戳索引文件的大小达到 broker 端参数 log.index.size.max.bytes配置的值。 log.index.size.max.bytes 的默认值为 10485760，即 10MB。
4. 切分索引文件预分配空间：索引文件会根据 log.index.size.max.bytes 值进行**预先分配空间**，即文件创建的时候就是最大值，当真正的进行索引文件切分的时候，才会将其裁剪到实际数据大小的文件。

## index文件

### **消息存储**

1. 消息内容保存在log日志文件中。
2. 消息封装为Record，追加到log日志文件末尾，采用的是**顺序写**模式。 
3. 一个topic的不同分区，可认为是queue，顺序写入接收到的消息。

### 查看index文件

```shell
[root@node1 tp_demo_05-0]# kafka-run-class.sh kafka.tools.DumpLogSegments --
files 00000000000003925423.log  --print-data-log | head
Dumping 00000000000003925423.log
Starting offset: 3925423
baseOffset: 3925423 lastOffset: 3926028 baseSequence: -1 lastSequence: -1
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional:
false position: 0 CreateTime: 1596513434779 isvalid: true size: 16359 magic:
2 compresscodec: NONE crc: 4049330741
baseOffset: 3926029 lastOffset: 3926634 baseSequence: -1 lastSequence: -1
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional:
false position: 16359 CreateTime: 1596513434786 isvalid: true size: 16359
magic: 2 compresscodec: NONE crc: 2290699169
baseOffset: 3926635 lastOffset: 3927240 baseSequence: -1 lastSequence: -1
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional:
false position: 32718 CreateTime: 1596513434787 isvalid: true size: 16359
magic: 2 compresscodec: NONE crc: 368995405
baseOffset: 3927241 lastOffset: 3927846 baseSequence: -1 lastSequence: -1
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional:
false position: 49077 CreateTime: 1596513434788 isvalid: true size: 16359
magic: 2 compresscodec: NONE crc: 143415655
baseOffset: 3927847 lastOffset: 3928452 baseSequence: -1 lastSequence: -1
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional:
false position: 65436 CreateTime: 1596513434789 isvalid: true size: 16359
magic: 2 compresscodec: NONE crc: 572340120
baseOffset: 3928453 lastOffset: 3929058 baseSequence: -1 lastSequence: -1
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional:
false position: 81795 CreateTime: 1596513434790 isvalid: true size: 16359
magic: 2 compresscodec: NONE crc: 1029643347
baseOffset: 3929059 lastOffset: 3929664 baseSequence: -1 lastSequence: -1
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional:
false position: 98154 CreateTime: 1596513434791 isvalid: true size: 16359
magic: 2 compresscodec: NONE crc: 2163818250
baseOffset: 3929665 lastOffset: 3930270 baseSequence: -1 lastSequence: -1
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional:
false position: 114513 CreateTime: 1596513434792 isvalid: true size: 16359
magic: 2 compresscodec: NONE crc: 3747213735
[root@node1 tp_demo_05-0]#
```

| 字段          | 含义                                                         | 重要性 |
| ------------- | ------------------------------------------------------------ | ------ |
| baseOffset    | 表示该分区的记录偏移量，指的是第几条记录                     | 重要‼️  |
| position      | 表示该记录在当前片段文件的文件偏移量                         | 重要‼️  |
| CreateTime    | 记录创建的时间                                               |        |
| isValid       | 记录是否有效                                                 |        |
| keysize       | 表示key的长度                                                |        |
| valuesize     | 表示value的长度                                              |        |
| magic         | 表示本次发布kafka服务程序协议的版本号                        |        |
| compresscodec | 压缩工具，None说明没有指定压缩类型，kafka目前提供了4种可选择，0-None、1- GZIP、2-snappy、3-lz4。 | 重要‼️  |
| sequence      | 消息的序列号                                                 | 重要‼️  |
| payload       | 表示具体的消息                                               |        |
| crc           | 对所有字段进行校验后的crc值。                                | 重要‼️  |

## 索引文件

**偏移量索引文件用于记录消息偏移量与物理地址之间的映射关系。**

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/image-20221205003707475.png" alt="image-20221205003707475" style="zoom:67%;" />

**时间戳索引文件则根据时间戳查找对应的偏移量。**

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/image-20221205003730453.png" alt="image-20221205003730453" style="zoom:67%;" />

### 偏移量索引文件的作用

通过相对偏移量找到日志的物理位置。Kafka 中的索引文件是以稀疏索引的方式构造消息的索引，并不保证每一个消息在索引文件中都有对应的索引项**，因为每当写入一定量4K(log.index.interval.bytes设定)的消息时，偏移量索引文件和时间戳索引文件分别增加一个偏移量索引项和时间戳索引项*，可以通过修改 log.index.interval.bytes 的值，改变索引项的密度。

### 时间戳索引文件的作用

可以让用户查询某个时间段内的消息，它一条数据的结构是时间戳(8byte)+相对offset(4byte)，如果要使用这个索引文件，首先需要通过时间范围找到对应的相对offset，然后再通过offset去对应的index文件找到position信息，然后才能遍历log文件，它也是需要使用index文件。

> 由于producer生产消息可以指定消息的时间戳，这可能将导致消息的时间戳不一定有先后顺序，因此**尽量不要生产消息时指定时间戳**。在 Kafka 0.11.0.0 以后，消息元数据中存在若干的时间 戳信息。如果 broker 端参数 log.message.timestamp.type 设置为 LogAppendTIme ，那么时间戳 必定能保持单调增长。反之如果是 CreateTime 则无法保证顺序。

## 日志的查找

### 偏移量索引查找

1. 根据offset二分查找segment对应的index文件（由于segment的文件名特性）
2. 找到index文件后，二分找到不大于offset的最大索引项，得到position
3. 根据position顺序扫描，找到索引对应的position。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/905646-20220927134001417-833945976.png)

如果要查找offset为4031802的消息，首先会通过二分找到对应的segment，然后去对应的index文件通过二分找到不大于4031802的最大索引项，也就是[offset: 4031789 position: 16906]，之后进行顺序扫描。即先通过二分法在.index文件找到offset为4031789 的那条，再从log文件中的物理偏移量为12605的位置开始顺序查找偏移量为4031802的消息

### 时间索引查找

1. 根据时间戳遍历segment对应的timeindex文件逐个比较，知道找到不小于该时间戳的segment（根据时间戳索引的文件名特性）
2. 找到segment的timeIndex文件后，二分找到不大于时间戳的最大索引项，得到偏移量。
3. 拿到偏移量之后，再根据上述说的*根据偏移量索引查找*进行数据检索。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/905646-20220927134018768-473464527.png)

如果要查找timstamp为1663899378988开始的消息，首先将时间戳和每个log文件中最大的时间戳largestTimeStamp进行逐一对比，直到找到不小于1663899378988所对应的segment。找到log文件后，使用二分法定位，找到不大于1663899378988的最大索引项，也就是[timestamp: 1663899378338 offset: 4031789]，拿着偏移量是4031789的offset值去偏移量索引文件找到不大于4031789的最大索引项，也就是[offset: 4031789 position: 16906]，之后从偏移量为16906的位置开始顺序查找，找到timestamp不小于1663899378988的消息。

## 参考

[kafka日志存储](https://www.cnblogs.com/kiko2014551511/p/16733479.html)

