# 核心概念

- Servers：kafka可运行在多可以跨机房的多节点上，这些节点成为broker
- Clients：用于并行读写events
- Event：key, value, timestamp和可选的metadata 组成的数据，类似于文件
- Topic & Partition ：分别是存储Events的逻辑和物理概念，topic类似于文件夹

- 并行：一个topic被partitioned到多个broker上，用于支持并行读写
- 写：相同key会被写到相同partition，同一个partition顺序写，一个event一个offset(整型值)，不同partition顺序不保证
- 读：同一个个partition按照写入的顺序读，不同partition顺序不保证
- 高可用：每个partition都会存在多个副本，分布在多个broker上，以达成高可用

- Producer：写Events到broker
- Consumer group &Comsumer：从brokers读Events，一个消费组由一个或多个消费者组成，便于扩容与容错，一个event只会被一个消费者消费，不同的消费组可以同时消费同一条消息。

# 核心机制 之 Rebalance

**原因**

1. 消费组发生了变更，比如有新的消费者加入了消费组或者有消费者宕机
2. 消费者无法在指定的时间之内完成消息的消费
3. Topic发生了变化
4. Topic的partition发生了变化

## Coordinator

- Coordinator：每个消费组都会有一个coordinator

- Coordinator负责管理消费组内的消费者和offset管理
- coordinator并不负责消费组内的partition分配
- 消费者通过心跳的方式告知Coordinator自己仍然处于存活状态

## Rebalance流程

Rebalance机制整体可以分为两个步骤，一个是Joining the Group，另外一个是分配Synchronizing Group State

- Joining the Group

- 在当前这个步骤中，所有的消费者会和Coordinator交互，请求加入当前消费组
- Coordinator会从所有的消费者中选择一个消费者作为leader consumer， 选择的算法是随机选择

- Synchronizing Group State

- leader Consumer从Coordinator获取所有的消费者的信息
- 将消费组订阅的partition分配结果封装为SyncGroup请求，发送给Coordinator
- Coordinator再将分配结果发送给各个Consumer。
- 分配partition有如下3种策略RangeAssignor，RoundRobinAssignor，StickyAssignor，

另外，如果leader consumer因为一些特殊原因导致分配分区失败(Coordinator通过超时的方式检测)，那么Coordinator会重新要求所有的Consumer重新进行步骤Joining the Group状态。

## Generation机制

- Generation：

- Coordinator每进行一次Rebalance，就会为当前的Rebalance设置一个Generation标记
- 消费者在提交offset的时候会将Generation一同提交
- Generation不匹配会被拒绝
- Generation的机制可能会导致上一Generation消费者和当前Generation消费者消费相同的消息，所以消费者需要实现消息消费的幂等性

## Leader Consumer

- Leader Consumer

- Leader Consumer负责组内各个Consumer的partition分配
- Leader Consumer还负责整个消费组订阅的主题的监控，会定期更新消费组订阅的主题信息，一旦发现主题信息发生变化，会通知Coordinator触发Rebalance机制。

# 核心机制 之 Exactly once

## 基本概念

- 至少一次语义(At least once semantics)：写确认，可能导致重复写，从而导致重复读。
- 至多一次语义(At most once semantics)：写不确认，可能导致漏写，从而导致漏读。
- 精确一次语义(Exactly once semantics)： 不会漏写，即使重复写，也只会读一次。

## 幂等 - Producer视角

**Producer的send操作是幂等的。**在任何导致producer重试的情况下，相同消息，即使被producer发送多次，也只会被写入Kafka一次。需要修改broker的配置来启用：**enable.idempotence = true。**

**概念**

- 幂等操作：是指可以执行多次，而不会产生与仅执行一次不同结果的操作
- ProducerID：在每个新的Producer初始化时，会被分配一个唯一的ProducerID，这个ProducerID对客户端使用者是不可见的
- SequenceNumber：对于每个ProducerID，Producer发送数据的每个Topic-Partition都对应一个从0开始单调递增的SequenceNumber值

**流程**

- 发送到Kafka的每批消息将包含一个ProducerID和SequenceNumber，用于检查数据是否重复
- ProducerID和SequenceNumber被持久化存储topic中，因此即使leader replica失败，接管的任何其他broker也将能感知到消息是否重复

## 事务 - Consumer视角

跨partition的原子性读+写操作，也即Producer将多条消息作为一个事务批量发送，要么全部成功要么全部失败。

典型场景是从Kafka中获取消息，再将处理完的数据写回Kafka的其它Topic中。为了实现该场景下的事务的原子性，Kafka需要保证对Consumer Offset的Commit与Producer对发送消息的Commit包含在同一个事务中。

否则，如果在二者Commit中间发生异常，根据二者Commit的顺序可能会造成数据丢失和数据重复：

- 如果先Commit Producer发送数据的事务再Commit Consumer的Offset，即At Least Once语义，可能造成数据重复。
- 如果先Commit Consumer的Offset，再Commit Producer数据发送事务，即At Most Once语义，可能造成数据丢失。