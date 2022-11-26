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