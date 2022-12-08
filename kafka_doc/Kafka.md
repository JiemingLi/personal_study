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

## [控制器](https://jiamaoxiang.top/2020/07/06/Kafka%E7%9A%84Controller-Broker%E6%98%AF%E4%BB%80%E4%B9%88/)

Kafka集群包含若干个broker，broker.id指定broker的编号，编号不要重复。

控制器就是一个broker，需要负责一些额外的工作(追踪集群中的其他Broker，并在合适的时候处理新加入的和失败的Broker节点、Rebalance分区、分配新的leader分区等)。

**Kafka集群中始终只有一个Controller Broker。**

### 作用

Controller Broker的主要职责有很多，主要是一些管理行为，主要包括以下几个方面：

- 创建、删除主题，增加分区并分配leader分区
- 集群Broker管理（新增 Broker、Broker 主动关闭、Broker 故障)
- **preferred leader**选举
- 分区重分配

### 如何选举

Broker 在启动时，会尝试去 ZooKeeper 中创建 /controller 节点。Kafka 当前选举控制器的规则是：**第一个成功创建 /controller 节点的 Broker 会被指定为控制器**。（利用zk临时节点的特性）

>  即:Kafka通过Zookeeper的分布式锁特性选举**集群控制器**。

### 处理broker下线的情况

当某个Broker节点由于故障离开Kafka群集时，则存在于该Broker的leader分区将不可用(由于客户端仅对leader分区进行读写操作)。为了最大程度地减少停机时间，需要快速找到替代的leader分区。

当控制器发现一个 broker 已经离开集群，那些失去Leader副本分区的Follower分区需要一个新 Leader(这些分区的首领刚好是在这个 broker 上)。

1. 控制器需要知道哪个broker宕机了?
   - 每个 Broker 启动后，会在zookeeper的 /Brokers/ids 下创建一个临时 znode。当 Broker 宕机或主动关闭后，该 Broker 与 ZooKeeper 的会话结束，这个 znode 会被自动删除。同理，ZooKeeper 的 Watch 机制将这一变更推送给控制器，这样控制器就能知道有 Broker 关闭或宕机了，控制器就知道哪些主题的leader分区宕机了，从而进行后续的协调操作。

2. 控制器需要知道宕机的broker上负责的时候哪些分区的Leader副本分区?
   - 每个 Broker 启动后，会在 zookeeper的 /Brokers/topics/具体的topic/partitions/分区号/state   下创建一个 znode，其可以看到某个主题的某个分区的情况，比如是否是leader，这个主题的isr是哪些等。

**整个过程：**

当某一台broker宕机后，控制器可以通过1知道是哪台服务器宕机，通过2知道宕机的有哪些主题的leader分区也宕机了，也知道其ISR列表，后续就决定哪些Broker上的分区成为leader分区，然后，它会通知每个相关的Broker，要么将Broker上的主题分区变成leader，要么通过`LeaderAndIsr`请求从新的leader分区中复制数据。

### 处理broker加入的情况

当Controller注意到Broker已加入集群时，它将使用Broker ID来检查该Broker上是否存在分区，如果存在，则Controller通知新加入的Broker和现有的Broker，新的Broker上面的follower分区再次开始复制现有leader分区的消息。为了保证负载均衡，Controller会将新加入的Broker上的follower分区选举为leader分区。

> **注意**：上面提到的选Leader分区，严格意义上是换Leader分区，为了达到负载均衡，可能会造成原来正常的Leader分区被强行变为follower分区。换一次 Leader 代价是很高的，原本向 Leader分区A(原Leader分区) 发送请求的所有客户端都要切换成向 B (新的Leader分区)发送请求，建议你在生产环境中把这个参数设置成 false。

### 脑裂

如果controller Broker 挂掉了，Kafka集群必须找到可以替代的controller，集群将不能正常运转。这里面存在一个问题，很难确定Broker是挂掉了，还是仅仅只是短暂性的故障。但是，集群为了正常运转，必须选出新的controller。如果之前被取代的controller又正常了，他并不知道自己已经被取代了，那么此时集群中会出现两台controller。

其实这种情况是很容易发生。比如，某个controller由于GC而被认为已经挂掉，并选择了一个新的controller。在GC的情况下，在最初的controller眼中，并没有改变任何东西，该Broker甚至不知道它已经暂停了。因此，它将继续充当当前controller，这是分布式系统中的常见情况，称为脑裂。

假如，处于活跃状态的controller进入了长时间的GC暂停。它的ZooKeeper会话过期了，之前注册的`/controller`节点被删除。集群中其他Broker会收到zookeeper的这一通知。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/%E8%84%91%E8%A3%821.png)

由于集群中必须存在一个controller Broker，所以现在每个Broker都试图尝试成为新的controller。假设Broker 2速度比较快，成为了最新的controller Broker。此时，每个Broker会收到Broker2成为新的controller的通知，由于Broker3正在进行”stop the world”的GC，可能不会收到Broker2成为最新的controller的通知。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/%E8%84%91%E8%A3%822.png)

等到Broker3的GC完成之后，仍会认为自己是集群的controller，在Broker3的眼中好像什么都没有发生一样。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/%E8%84%91%E8%A3%823.png)

现在，集群中出现了两个controller，它们可能一起发出具有冲突的命令，就会出现脑裂的现象。如果对这种情况不加以处理，可能会导致严重的不一致。所以需要一种方法来区分谁是集群当前最新的Controller。

Kafka是通过使用**epoch number**（纪元编号，也称为隔离令牌）来完成的。epoch number只是单调递增的数字，第一次选出Controller时，epoch number值为1，如果再次选出新的Controller，则epoch number将为2，依次单调递增。

每个新选出的controller通过Zookeeper 的条件递增操作获得一个全新的、数值更大的epoch number 。其他Broker 在知道当前epoch number 后，如果收到由controller发出的包含较旧(较小)epoch number的消息，就会忽略它们，即Broker根据最大的epoch number来区分当前最新的controller。

![img](https://gitee.com/JieMingLi/document-pics/raw/master/%E8%84%91%E8%A3%824.png)

上图，Broker3向Broker1发出命令:让Broker1上的某个分区副本成为leader，该消息的epoch number值为1。于此同时，Broker2也向Broker1发送了相同的命令，不同的是，该消息的epoch number值为2，此时Broker1只听从Broker2的命令(由于其epoch number较大)，会忽略Broker3的命令，从而避免脑裂的发生。

### 结论

1. Kafka使用 Zookeeper 的分布式锁选举控制器，并在节点加入集群或退出集群时通知控制器。
2. 控制器负责在节点加入或离开集群时进行分区Leader选举。
3. 控制器使用epoch来避免“脑裂”。“脑裂”是指两个节点同时认为自己是当前的控制器。

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

### 参考

[kafka日志存储](https://www.cnblogs.com/kiko2014551511/p/16733479.html)

## 日志清理

Kafka 提供两种日志清理策略:

- 日志删除:按照一定的删除策略，将不满足条件的数据进行数据删除

- 日志压缩:针对每个消息的 Key 进行整合，对于有相同 Key 的不同 Value 值，只保留最后一个版本。

> Kafka 提供 log.cleanup.policy 参数进行相应配置，默认值: delete ，还可以选择 compact 。

### 删除

**基于时间删除**：根据时间戳索引文件找到第一条比需要删除时间小的最大时间戳，索引到对应的日志。

1. 从日志对象中所维护日志分段的跳跃表中移除待删除的日志分段，保证没有线程对这些日志分段进行读取操作（避免线程冲突）。
2. 这些日志分段所有文件添加 上 .delete 后缀。
3. 交由一个以 "delete-file" 命名的延迟任务来删除这些 .delete 为后缀的文件。延迟执行时间可以通过 file.delete.delay.ms 进行设置

>   **如果活跃的日志分段中也存在需要删除的数据时?**
>
> *Kafka 会先切分出一个新的日志分段作为活跃日志分段，该日志分段不删除，删除原来的日志分 段。*

**基于日志大小删除**：日志删除任务会检查当前日志的大小是否超过设定值。设定项为 log.retention.bytes ，单个日 志分段的大小由 log.segment.bytes 进行设定。

1. 计算需要被删除的日志总大小
2.  从日志文件第一个 LogSegment 开始查找可删除的日志分段的文件集合。
3.  执行删除。

**基于偏移量删除**：根据日志分段的**下一个**日志分段的起始偏移量是否大于等于日志文件的起始偏移量，若是，则可以删除此日志分段。

![image-20221206165855555](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221206165855555.png)

1. 从头开始遍历每个日志分段，日志分段1的下一个日志分段的起始偏移量为21，小于 logStartOffset，将日志分段1加入到**删除队列**中
2. 日志分段 2 的下一个日志分段的起始偏移量为35，小于 logStartOffset，将 日志分段 2 加入 到删除队列中
3. 日志分段 3 的下一个日志分段的起始偏移量为57，小于logStartOffset，将日志分段3加入删除 集合中
4. 日志分段4的下一个日志分段的其实偏移量为71，大于logStartOffset，则不进行删除。

### 压缩

1. 对于具有相同的Key，而数据不同，只保留最后一条数据，前面的数据在合适的情况下删除。
2. 日志压缩与key有关，确保每个消息的**key不为null**。

**应用场景**

在Spark、 Flink中做实时计算时，需要长期在内存里面维护一些数据，这些数据可能是通过聚合了一天或者一周的 日志得到的，这些数据一旦由于异常因素(内存、网络、磁盘等)崩溃了，从头开始计算需要很长的时 间。一个比较有效可行的方式就是定时将内存里的数据备份到外部存储介质中，当崩溃出现时，再从外 部存储介质中恢复并继续计算。

*使用Kafka的压缩存储会有什么好处？*

- Kafka即是数据源又是存储工具，可以简化技术栈，降低维护成本
- 使用外部存储介质的话，需要将存储的Key记录下来，恢复的时候再使用这些Key将数据取 回，实现起来有一定的工程难度和复杂度。使用Kafka的日志压缩特性，只需要把数据写进 Kafka，等异常出现恢复任务时再读回到内存就可以了（可以对比redis和kafka的区别）
- Kafka对于磁盘的读写做了大量的优化工作，比如磁盘顺序读写。相对于外部存储介质没有索 引查询等工作量的负担，可以实现高性能。同时，Kafka的日志压缩机制可以充分利用廉价的 磁盘，不用依赖昂贵的内存来处理，在性能相似的情况下，实现非常高的性价比(这个观点仅 仅针对于异常处理和容灾的场景来说)

**过程**

主题的 cleanup.policy 需要设置为compact。

Kafka的后台线程会定时将Topic遍历2次

-  记录每个key的hash值最后一次出现的偏移量
-  第2次检查每个offset对应的Key是否在后面的日志中出现过，如果出现了就删除对应的日志。

日志压缩允许删除，除最后一个key之外，删除先前出现的所有该key对应的记录（添加删除标识）。在一段时间后从日志中清理，以释放空间。**Kafka 提供了专门的后台线程定期地巡检待 Compact 的主题，看看是否存在满足条件的可删除数据**。



压缩是在Kafka后台通过**定时重新打开**Segment来完成

![image-20221206204419199](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221206204419199.png)

虽然日志压缩，但可以依然可以保留一些特性

- 消息始终保持顺序，压缩永远不会重新排序消息，只是删除一些而已
- 消息的偏移量永远不会改变，它是日志中位置的永久标识符
- 日志被删除后，日志可以保留Topic的log.cleaner.delete.retention.ms短的时间，默认是24小时，但是被压缩的一些日志会有delete标记。

Kafka的日志压缩定时把所有的日志读取两遍，写一遍，而CPU的速度超过磁盘完全不是问题，只要日志的量对应的读取两遍和写入一遍的时间在可接受的范围内，那么它的性能就是 可以接受的。

## 零拷贝，页缓存

### DMA

**在进行 I/O 设备和内存的数据传输的时候，数据搬运的工作全部交给 DMA 控制器，而 CPU 不再参与任何与数据搬运相关的事情，这样 CPU 就可以去处理别的事务**。

![image-20221207131313934](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221207131313934.png)

原本计算机所有组件之间的数据拷贝（流动）必须经过 CPU

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/c9e57c79d51c3bc86595dcec3ee097b4.png" alt="2.png" style="zoom:67%;" />

现在，DMA 代替了 CPU 负责内存与磁盘以及内存与网卡之间的数据搬运，CPU 作为 DMA 的控制者，如下图所示：

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/f53083abb1b57ac58d9f10be511cd406.png" alt="3.png" style="zoom:67%;" />

但是 DMA 有其局限性，DMA 仅仅能用于设备之间交换数据时进行数据拷贝，但是设备内部的数据拷贝还需要 CPU 进行。

### 零拷贝

**传统的数据传输方式**

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/%E4%BC%A0%E7%BB%9F%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93.png" alt="img" style="zoom:67%;" />

期间共**发生了 4 次用户态与内核态的上下文切换**，因为发生了两次系统调用，一次是 `read()` ，一次是 `write()`，每次系统调用都得先从用户态切换到内核态，等内核完成任务后，再从内核态切换回用户态。

> 上下文切换到成本并不小，一次切换需要耗时几十纳秒到几微秒，虽然时间看上去很短，但是在高并发的场景下，这类时间容易被累积和放大，从而影响系统的性能。

**发生了 4 次数据拷贝**，其中两次是 DMA 的拷贝（如果没有DMA还需要cpu自己拷贝），另外两次则是通过 CPU 拷贝的

- *第一次拷贝*，把磁盘上的数据拷贝到操作系统内核的缓冲区里，这个拷贝的过程是通过 DMA 搬运的。
- *第二次拷贝*，把内核缓冲区的数据拷贝到用户的缓冲区里，于是我们应用程序就可以使用这部分数据了，这个拷贝到过程是由 CPU 完成的。
- *第三次拷贝*，把刚才拷贝到用户的缓冲区里的数据，再拷贝到内核的 socket 的缓冲区里，这个过程依然还是由 CPU 搬运的。
- *第四次拷贝*，把内核的 socket 缓冲区里的数据，拷贝到网卡的缓冲区里，这个过程又是由 DMA 搬运的。

> 搬运一份数据，结果却搬运了 4 次，过多的数据拷贝无疑会消耗 CPU 资源，大大降低了系统性能,，简单的数据传输，没必要进行过多的拷贝。

**为什么kafka使用0拷贝**

由上可知，**想提高文件传输的性能，就需要减少「用户态与内核态的上下文切换」和「内存拷贝」的次数**。用户进程在数据传输过程中并不需要对数据进行访问和处理，没必要再把数据从内核空间（page cache）拷贝到用户空间（用户缓存），让数据在内核里面完成拷贝即可，而且数据最好是小文件（消息不会太大，所以更加适用于0拷贝）

**0拷贝的2种实现方式**

1. **mmap（减少内核空间到用户空间的数据拷贝） + write**

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/mmap.jpg" alt="mmap" style="zoom:67%;" />

- 用户进程调用 `mmap()`，从用户态陷入内核态，将内核缓冲区映射到用户缓存区；

- DMA 控制器将数据从硬盘拷贝到内核缓冲区（可见其使用了 Page Cache 机制）；
- `mmap()` 返回，上下文从内核态切换回用户态；
- 用户进程调用 `write()`，尝试把文件数据写到内核里的套接字缓冲区，再次陷入内核态；
- CPU 将内核缓冲区中的数据拷贝到的套接字缓冲区；
- DMA 控制器将数据从套接字缓冲区拷贝到网卡完成数据传输；
- `write()` 返回，上下文从内核态切换回用户态。

2. **sendFile**

使用sendfile代替传统的read 和write系统调用，把2次系统调用改为1次，也就减少了 2 次上下文切换的开销。sendfile可以直接把内核缓冲区里的数据拷贝到 socket 缓冲区里，不再拷贝到用户态，这样就只有 2 次上下文切换，和 3 次数据拷贝。如下图：

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/senfile-3%E6%AC%A1%E6%8B%B7%E8%B4%9D.png" alt="img" style="zoom:67%;" />



如果网卡支持 SG-DMA（*The Scatter-Gather Direct Memory Access*）技术（和普通的 DMA 有所不同），我们可以进一步减少通过 CPU 把内核缓冲区里的数据拷贝到 socket 缓冲区的过程（从 Linux 内核 `2.4` 版本开始起，对于支持网卡支持 SG-DMA 技术的情况下）

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/senfile-%E9%9B%B6%E6%8B%B7%E8%B4%9D.png" alt="img" style="zoom:67%;" />

- 第一步，通过 DMA 将磁盘上的数据拷贝到内核缓冲区里；
- 第二步，缓冲区描述符和数据长度传到 socket 缓冲区，这样网卡的 SG-DMA 控制器就可以直接将内核缓存中的数据拷贝到网卡的缓冲区里，此过程不需要将数据从操作系统内核缓冲区拷贝到 socket 缓冲区中，这样就减少了一次数据拷贝；

> 零拷贝（*Zero-copy*）技术，因为我们没有在内存层面去拷贝数据，也就是说全程没有通过 CPU 来搬运数据，所有的数据都是通过 DMA 来进行传输的

**0拷贝的优点：**

相比传统文件传输的方式，减少了 2 次上下文切换和数据拷贝次数，只需要 2 次上下文切换和数据拷贝次数，就可以完成文件的传输，而且 2 次的数据拷贝过程，都不需要通过 CPU，2 次都是由 DMA 来搬运。

**kafka的两个过程:**

1. 网络数据持久化到磁盘 (Producer 到 Broker) （broker读取到producer得数据，使用mmap机制进行写操作，顺序写入磁盘。）
2. 磁盘文件通过网络发送(Broker 到 Consumer)（sendfile的机制实现0拷贝，发送数据给consumer）

> 数据落盘通常都是非实时的，Kafka的数据并不是实时的写入硬盘，它充分利用了现代操作系统分页存储（写入page cache）来利用内存提高I/O效率。

**kafka最终通过Java NIO最终调用到sendfile**

Kafka 文件传输的代码最终它调用了 Java NIO 库里的 `transferTo` 方法：

```java
@Overridepublic 
long transferFrom(FileChannel fileChannel, long position, long count) throws IOException { 
    return fileChannel.transferTo(position, count, socketChannel);
}
```

如果 Linux 系统支持 `sendfile()` 系统调用，那么 `transferTo()` 实际上最后就会使用到 `sendfile()` 系统调用函数。

**kafka可以使用0拷贝的前提：**

- 系统支持sendfile（Linux 内核版本必须要 2.1 以上的版本）
- 如果需要cpu完全不拷贝数据，则需要网卡支持 SG-DMA（Linux 内核版本必须要 2.4 以上的版本）

### 页缓存和mmap

##### **页缓存**

**消息先被写入页缓存，由操作系统负责刷盘任务。**

页缓存是操作系统实现的一种主要的磁盘缓存，以此用来减少对磁盘 I/O 的操作，把磁盘中的数据缓存到内存中，把对磁盘的访问变为对内存的访问。

Kafka接收来自socket buffer的网络数据，应用进程不需要中间处理、直接进行持久化时。可以使用mmap内存文件映射。（相当于把read替换为mmap）

**不适合读取大文件数据到page cahce**

在传输大文件（GB 级别的文件）的时候，PageCache 会不起作用，那就白白浪费 DMA 多做的一次数据拷贝，造成性能的降低，即使使用了 PageCache 的零拷贝也会损失性能

因为如果你有很多 GB 级别文件需要传输，每当用户访问这些大文件的时候，内核就会把它们载入 PageCache 中，于是 PageCache 空间很快被这些大文件占满。

带来以下2个问题

- PageCache 由于长时间被大文件占据，其他「热点」的小文件可能就无法充分使用到 PageCache，于是这样磁盘读写的性能就会下降了；
- PageCache 中的大文件数据，由于没有享受到缓存带来的好处，但却耗费 DMA 多拷贝到 PageCache 一次；

> 针对大文件的传输，不应该使用 PageCache，也就是说不应该使用零拷贝技术，因为可能由于 PageCache 被大文件占据，而导致「热点」小文件无法利用到 PageCache，这样在高并发的环境下，会带来严重的性能问题。

**场景**

读取大文件适合用异步IO + 直接IO（看下文）

kafka传送消息不属于大文件范畴，所以适用于零拷贝

[大文件合适使用的场景](https://xiaolincoding.com/os/8_network_system/zero_copy.html#%E5%A4%A7%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93%E7%94%A8%E4%BB%80%E4%B9%88%E6%96%B9%E5%BC%8F%E5%AE%9E%E7%8E%B0)

##### **mmap**

mmap：将磁盘文件映射到内存, 用户通过修改内存就能修改磁盘文件，直接利用操作系统的Page来实现磁盘文件到物理内存的直接映射。完成映射之后你 对物理内存的操作会被同步到硬盘上(操作系统在适当的时候)

使用mmap的好处：通过mmap，进程像读写硬盘一样读写内存(当然是虚拟机内存)。使用这种方式可以获取很大的 I/O提升，**省去了用户空间到内核空间复制的开销**。

使用mmap的缺点：:不可靠，写到mmap中的数据并没有被真正的写到硬盘，操作系统 会在程序主动调用flush的时候才把数据真正的写到硬盘。

>  当使用mmap把数据从磁盘读取到page cache 的时候，用户进程读写数据的过程如下（可以类比mysql的写盘机制）

当一个进程准备读取磁盘上的文件内容时：

1. 操作系统会先查看待读取的数据所在的页 (page)是否在页缓存(pagecache)中，如果存在(命中) 则直接返回数据，从而避免了对物理磁盘的 I/O 操作;
2. 如果没有命中，则操作系统会向磁盘发起读取请求并将读取的数据页存入页缓存，之后再将数 据返回给进程。

当一个进程需要将数据写入磁盘:

1. 操作系统也会检测数据对应的页是否在页缓存中，如果不存在，则会先在页缓存中添加相应的 页，最后将数据写入对应的页。
2. 被修改过后的页也就变成了脏页，操作系统会在合适的时间把脏页中的数据写入磁盘，以保持 数据的一致性。

*Kafka中大量使用了页缓存，这是 Kafka 实现高吞吐的重要因素之一。*

##### 直接IO

[看参考3.4](https://spongecaptain.cool/SimpleClearFileIO/2.%20DMA%20%E4%B8%8E%E9%9B%B6%E6%8B%B7%E8%B4%9D%E6%8A%80%E6%9C%AF.html)

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/directIO.jpg" alt="directIO" style="zoom:67%;" />

### 参考

[深入理解零拷贝技术](http://dockone.io/article/2434459)

[mmap代码了解](https://github.com/Spongecaptain/SimpleClearFileIO/issues/1)

[什么是0拷贝-小林](https://xiaolincoding.com/os/8_network_system/zero_copy.html)

[mmap基本概念](https://spongecaptain.cool/SimpleClearFileIO/3.%20mmap.html)

[DMA和0拷贝技术](https://spongecaptain.cool/SimpleClearFileIO/2.%20DMA%20%E4%B8%8E%E9%9B%B6%E6%8B%B7%E8%B4%9D%E6%8A%80%E6%9C%AF.html)

## 顺序读写

操作系统可以针对线性读写做深层次的优化（空间局部性），比如预读(read-ahead，提前将一个比较大的磁盘块读入内存，定位到偏移量之后，后续的也可以很快读取到了，减少后续有随机读写的概率) 和后写(write-behind，将很多小的逻辑写操作合并起来组成一个大的物理写操作，顺序写，避免随机磁盘io，也保障读取的时候可以顺序读)技术。

<img src="https://gitee.com/JieMingLi/document-pics/raw/master/image-20221207133253867.png" alt="image-20221207133253867" style="zoom:67%;" />

Kafka 在设计时采用了**文件追加**的方式来写入消息，即只能在日志文件的尾部追加新的消息，**并且也不允许修改已写入的消息**（压缩或者删除），这种方式属于典型的顺序写盘的操作，所以就算 Kafka 使用磁盘作为存储介质，也能承载非常大的吞吐量。

> 硬盘是机械结构，每次读写都会寻址->写入，其中寻址是一个“机械动作”，它是最耗时的。所以硬盘最讨厌随机I/O，最喜欢顺序I/O。为了提高读写硬盘的速度，Kafka就是使用顺序I/O。这样省去了大量的内存开销以及节省了IO寻址的时间。

## kafka处理速度快的原因

1. partition顺序读写，充分利用磁盘特性，这是基础;
2. Producer生产的数据持久化到broker，采用mmap文件映射，实现顺序的快速写入;
3. Customer从broker读取数据，采用sendfile，将磁盘文件读到OS内核缓冲区后，直接转到socket buffer进行网络发送。

# 可靠性保证

副本因子，副本节点，ISR，OSR，副本的分配

![image-20221207221058374](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221207221058374.png)

# 一致性保证

HW， Leo， Remote Leo，Leader epoch 

[高水位，leader epoch](https://www.cnblogs.com/huxi2b/p/7453543.html)

[高水位](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Kafka%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%AE%9E%E6%88%98/27%20%20%E5%85%B3%E4%BA%8E%E9%AB%98%E6%B0%B4%E4%BD%8D%E5%92%8CLeader%20Epoch%E7%9A%84%E8%AE%A8%E8%AE%BA.md)

# 消息重复的场景和解决方案

发生重试的场景

1. 生产者阶段 
2. broke阶段 
3. 消费者阶段

## 生产者重复发送

生产发送的消息没有收到正确的broke响应，导致生产者重试；生产者发出一条消息，broke落盘以后因为网络等种种原因发送端得到一个发送失败的响应或者网络中断，然后生产者收到一个可恢复的Exception重试消息导致消息重复。

**过程**

![image-20221208153527228](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221208153527228.png)

1. new KafkaProducer()后创建一个后台线程KafkaThread扫描RecordAccumulator中是否有消 息;
2. 调用KafkaProducer.send()发送消息，实际上只是把消息保存到RecordAccumulator中;
3. 后台线程KafkaThread扫描到RecordAccumulator中有消息后，将消息发送到kafka集群;
4. 如果发送成功，那么返回成功;
5. 如果发送失败，那么判断是否允许重试。如果不允许重试，那么返回失败的结果;如果允许重试，把消息再保存到RecordAccumulator中，等待后台线程KafkaThread扫描再次发送;

### 解决方案

要启动kafka的幂等性，设置: enable.idempotence=true ，以及 ack=all 以及 retries > 1 。

## 生产者丢失数据的场景

**ack = 0，不重试**

生产者发送消息完，不管结果了，如果发送失败也就丢失了。

**Ack = 1 （只有leader收到才返回），leader 崩溃**

生产者发送消息完，只等待Leader写入成功就返回了，Leader分区丢失了，此时Follower没来及同步，消息丢失。

**unclean.leader.election.enable = true**

允许选举ISR以外的副本作为leader,会导致数据丢失，默认为false。生产者发送异步消息，只等待Lead写入成功就返回，Leader分区丢失，此时ISR中没有Follower，Leader从OSR中选举，因为OSR中本来落后于Leader造成消息丢失。

### 解决方案

**禁用unclean选举，并且ack=all**

1， *ack=all / -1,tries > 1*

生产者发完消息，等待Follower同步完再返回，如果异常则重试。副本的数量可能影响吞吐量，不超过5个，一般三个。

2， *unclean.leader.election.enable : false*

不允许unclean Leader选举。

**配置:min.insync.replicas > 1**

当生产者将 acks 设置为 all (或 -1 )时， min.insync.replicas>1 。指定确认消息写成功需要的最小副本数量。达不到这个最小值，生产者将引发一个异常(要么是NotEnoughReplicas，要么是NotEnoughReplicasAfterAppend)。

> 当一起使用时， min.insync.replicas 和 ack 允许执行更大的持久性保证。一个典型的场景是创建一个复制因子为3的主题，设置min.insync复制到2个，用 all 配置发送。将确保如果大多数副本没有收到写操作，则生产者将引发异常。

**失败的offset单独记录**

生产者发送消息，会自动重试，遇到不可恢复异常会抛出，这时可以捕获异常记录到数据库或缓存，进行单独处理。

## 生产者发送的顺序

如果设置 `max.in.flight.requests.per.connection` 大于1(默认5，单个连接上发送的未确认 请求的最大数量，表示上一个发出的请求没有确认下一个请求又发出了)。大于1可能会改变记录的顺 序，因为如果将两个batch发送到单个分区，第一个batch处理失败并重试，但是第二个batch处理成 功，那么第二个batch处理中的记录可能先出现被消费。

设置 `max.in.flight.requests.per.connection` 为1，可能会影响吞吐量，可以解决**单个生产者**发送顺序问题，多个生产者的顺序还是无法保证，比如如果多个生产者，生产者1先发送一个请求，生产者2后发送请求，此时生产者1返回可 恢复异常，重试一定次数成功了。虽然生产者1先发送消息，但生产者2发送的消息会被先消费。

## 消费者重复消费

**原因**

数据消费完没有及时提交offset到broker。

**场景**

消息消费端在消费过程中挂掉没有及时提交offset到broke，另一个消费端启动拿之前记录的offset开始消费，由于offset的滞后性可能会导致新启动的客户端有少量重复消费。

### 解决方案

**取消自动提交**

每次消费完或者程序退出时手动提交。这可能也没法保证一条重复。

**下游做幂等**

一般是让下游做幂等或者尽量每消费一条消息都记录offset

1， 对于少数严格的场景可能需要把offset 或唯一ID(例如订单ID)和下游状态更新放在同一个数据库里面做事务来保证精确的一次更新

2， 在下游数据表里面同时记录消费offset，然后更新下游数据的时候用消费位移做乐观锁拒绝旧位移的数据更新。

# consumer_offset

[参考](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Kafka%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%AE%9E%E6%88%98/16%20%20%E6%8F%AD%E5%BC%80%E7%A5%9E%E7%A7%98%E7%9A%84%E2%80%9C%E4%BD%8D%E7%A7%BB%E4%B8%BB%E9%A2%98%E2%80%9D%E9%9D%A2%E7%BA%B1.md)

Zookeeper不适合大批量的频繁写入操作（为什么zk不适合频繁写操作？）

Kafka 1.0.2将consumer的位移信息保存在Kafka内部的topic中，即__consumer_offsets主题，并且默认提供了kafka_consumer_groups.sh脚本供用户查看consumer信息。

**当 Kafka 集群中的第一个 Consumer 程序启动时，Kafka 会自动创建位移主题**。

Broker 端参数 offsets.topic.num.partitions 的取值了。它的默认值是 50，因此 Kafka 会自动创建一个 50 分区的位移主题。

当消费者进行消费消息提交位移时，会像一个生产者一样，把消费到的位移发送给broker的comsumer_offset主题（这个消息的格式有key值，key：<groupId：topic：消费到的分区号>，value是位移值）因为comsumer_offset默认有50个分区，所以需要确认这个消费者组的位移消息发送到哪个comsumer_offset的哪个分区，这个分区的计算规则就是：`Math.abs(groupId.hashCode()) % 50` ，假如计算出来的值是15，则要把这个消费者组的所有的位移信息发送到comsumer_offset的15号分区，找到15号分区的leader副本，包含这个leader副本的broker就是这个主题的**协调者**（可以关联到消费者组的管理），后续向其传输位移数据。

问题：自动提交和手动提交的详解

# 延时队列

备注：需要了解延时队列在业务场景的使用。

场景：火车订单30s内未支付，则自动取消订单，https://juejin.cn/post/7065664161786626084

延时队列的其他实现：RocketMQ，Redis，XXX-JOB，JUC的代码

## **背景**

两个follower副本都已经拉取到了leader副本的最新位置，此时又向leader副本发送拉取请求，而 leader副本并没有新的消息写入，那么此时leader副本该如何处理呢?可以直接返回空的拉取结果给 follower副本，不过在leader副本一直没有新消息写入的情况下，follower副本会一直发送拉取请求，并 且总收到空的拉取结果，消耗资源。

## 拉取消息延时

Kafka在处理拉取请求时，会先读取一次日志文件，如果收集不到足够多(fetchMinBytes，由参数 `fetch.min.bytes`配置，默认值为1)的消息，那么就会创建一个延时拉取操作(DelayedFetch)以等待 拉取到足够数量的消息。当延时拉取操作执行时，会再读取一次日志文件，然后将拉取结果返回给 follower副本。

> 在Kafka中有多种延时操作，比如延时数据删除、延时生产等。

## 延时生产

如果在使用生产者客户端发送消息的时候将acks参数设置为-1，那么 就意味着需要等待ISR集合中的所有副本都确认收到消息之后才能正确地收到响应的结果，或者捕获超时异常。

![image-20221208182430378](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221208182430378.png)

由于客户端设置了acks为-1，那么需要等到follower1和follower2两个副本都收到消息3和消息4后 才能告知客户端正确地接收了所发送的消息。如果在一定的时间内，follower1副本或follower2副本没 能够完全拉取到消息3和消息4，那么就需要返回超时异常给客户端。生产请求的超时时间由参数 request.timeout.ms配置，默认值为30000，即30s。

![image-20221208182520234](https://gitee.com/JieMingLi/document-pics/raw/master/image-20221208182520234.png)

**这里等待消息3和消息4写入follower1副本和follower2副本，并返回相应的响应结果给客户端 的动作是由谁来执行？**

在将消息写入leader副本的本地日志文件之后，Kafka会创建一个延时的生产操作(DelayedProduce)

延时操作：*用来处理消息正常写入所有副本或超时的情况，以返回相应的响应结果给客户端。*

延时操作需要延时返回响应的结果，首先它必须有一个**超时时间**(delayMs)，

1， 如果在这个超时时间内没有完成既定的任务，那么就需要强制完成以返回响应结果给客户端。

2， 延时操作不同于定时操作，定时操作是指在特定时间之后执行的操作，而延时操作可以在所设定的超时时间之前完成，所以延时操作能够**支持外部事件的触发**。

**什么是外部事件？**

外部事件就是所要写入消息的某个分区的HW(高水位)发生增长。也就 是说，随着follower副本不断地与leader副本进行消息同步，进而促使HW进一步增长，HW每增长一次 都会检测是否能够完成此次延时生产操作，如果可以就执行以此返回响应结果给客户端;如果在超时时 间内始终无法完成，则强制执行。

## 延时拉取

延时拉取操作是由**超时触发**或外部事件触发而被执行。

**超时触发**

就是等到超时时间 之后触发第二次读取日志文件的操作。

**外部事件**

拉取请求不单单由follower 副本发起，也可以由消费者客户端发起，两种情况所对应的外部事件也是不同的。

1. 如果是follower副本的延时拉取，它的外部事件就是消息追加到了leader副本的本地日志文件中。
2. 如果是消费者客户端的延时拉取，它的外部事件可以简单地理解为HW的增长。

## 参考

https://www.cnblogs.com/fanchengmeng/p/16362441.html

[kafka实现延时队列](https://it-blog-cn.com/blogs/qmq/queue.html)

## 实现

[时间轮算法]((https://juejin.cn/post/6844904110399946766))

> 简单了解即可，太过于复杂。



# 重试队列和死信队列

死信可以看作消费者不能处理收到的消息，也可以看作消费者不想处理收到的消息，还可以看作不符合处理要求的消息。比如消息内包含的消息内容无法被消费者解析，为了确保消息的可靠性而不被随意丢弃，故将其投递到死信队列中，这里的死信就可以看作消费者不能处理的消息。再比如超过既定的重试次数之后将消息投入死信队列，这里就可以将死信看作不符合处理要求的消息。

进入死信队列可以由人工进行处理

如果消费者消费多次后都失败（期间也可以进行延迟），满足一定的阈值之后，再放入死信队列进行人工处理。

## 实现

kafka没有重试机制不支持消息重试，也没有死信队列，因此使用kafka做消息队列时，需要自己实现消息重试的功能。

创建新的kafka主题作为重试队列:

1. 创建一个topic作为重试topic，用于接收等待重试的消息。
2. 在消费者消费消息失败的时候，把需要重试的消息发送给重试topic。
3. 从重试topic获取待重试消息储存到redis的zset中，并以下一次消费时间排序 （redis的zset使用的场景）
4. 定时任务（spring的@Scheduled）从redis获取到达消费事件的消息，并把消息发送到对应的topic，进行再次消费。
5. 同一个消息重试次数过多则不再重试（第1次10s，第2次30s等），重试次数过多则放入死信队列。

