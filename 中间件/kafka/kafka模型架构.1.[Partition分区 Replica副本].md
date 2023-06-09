# kafka模型架构[Partition分区 Replica副本]

参考文章

1. [kafka partition分配_震惊了！原来这才是 Kafka！（多图+深入）](https://blog.csdn.net/weixin_42310891/article/details/112264408)
    - 全面
    - 第1张图, topic0分为2个分区, 每个分区有3个副本(一主两从).
    - 同一个消费组中的两个消费者, 不会消费同一个partition
    - 关于`offset`的解释, 很详细
2. [「Kafka深度解析」快速入门](https://www.jianshu.com/p/da1222dd0d32)
    - kafka特性
    - kafka核心概念
    - kafka**消费者**在会保存其消费的进度, 也就是offset
3. [Java工程师的进阶之路 Kafka篇（一）](https://www.jianshu.com/p/cbf684893574)
    - 消息系统存在必要性: 解耦, 冗余, 扩展性, 灵活性&峰值处理能力, 可恢复性, 顺序保证, 缓冲, 异步通信
    - ~~名词概念的解释并不是很清晰~~
    - 设计思想(很不错)
    - 应用场景(不错)
    - Push 模式 vs Pull 模式(非常不错!)
4. [kafka partition（分区）与 group](https://www.cnblogs.com/liuwei6/p/6900686.html)
    - 对于传统的message queue而言, 一般会删除已经被消费的消息, 而Kafka集群会保留所有的消息, 无论其被消费与否. 
    - 当然, 因为磁盘限制, 不可能永久保留所有数据（实际上也没必要）, 因此Kafka提供两种策略删除旧数据. 一是基于时间, 二是基于Partition文件大小. 
    - Kafka读取特定消息的时间复杂度为O(1), 即与文件大小无关, 所以这里删除过期文件与提高Kafka性能无关. 选择怎样的删除策略只与磁盘以及具体的需求有关

## 基本架构

消息系统中按`topic`进行发布订阅是比较常见的解决方案, 这里不再解释`topic`的概念.

除了`topic`, kafka中还有`partition(分区)`, `replica(副本)`以及消费组等术语, 可见参考文章3...但是这些看一遍基本是没法看懂的(比如我), 所以本文尽量用通俗的语言解释其机制.

## partition(分区)

首先`topic`逻辑对象为一个(有序)队列, 一般的场景就是, 生产者向队列尾部添加消息, 消费者从队列头获取消息.

如果单个`topic`的队列过长, 数据过多, 单个主机可能无法承受, 为了实现高可用及横向扩展能力, 会将`topic`数据进行**分区**. 

假设存在一个`topic`, 名为`topic0`, 其全部内容为: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13], 在一个3主机的kafka集群中, kafka可以将些`topic`划分为3个分区, 每台主机拥有1个分区.

- topic0-partition0: [1, 3, 7, 10]
- topic0-partition1: [2, 5, 9, 12, 13]
- topic0-partition2: [4, 6, 8, 11]

这样, 一个kafka集群就可以容纳3倍于单台主机单个kafka实例的数据量, 实现了横向扩展.

## replica(副本)

上面只是实现了了横向扩展, 如果其中某个kafka实例所在主机异常宕机, 那么上面的分区就会丢失. 为了实现高可用, kafka可以为每个分区创建n个副本. 在上面的场景中, 我们假设集群中有3台主机, 那么每个分区可以拥有3个副本(一主两从).

```
   host0          host1          host2   
+---------+    +---------+    +---------+
|   P0    |    |   P1    |    |   P2    |
|         |    |         |    |         |
|   R11   |    |   R01   |    |   R12   |
|         |    |         |    |         |
|   R21   |    |   R22   |    |   R02   |
+---------+    +---------+    +---------+
                                         
```

其中分区P与副本R的对应关系如下

- `P0`: `R01`+`R02`, 存储着[1, 3, 7, 10]
- `P1`: `R11`+`R12`, 存储着[2, 5, 9, 12, 13]
- `P2`: `R21`+`R22`, 存储着[4, 6, 8, 11]

## Leader与Follower

从概念上说, `PX`, `RXX`...其实都是副本, `P0`, `R01`, `R02`ta们3个整体被当作分区0.

不过`P0`, `P1`, `P2`称为`Leader`, `RXX`称为`Follower`, 当然, ta们之间的关系是可以相互转换的.

接入kafka集群的客户端(包括生产者与消费者)对数据的读写, 都是在`Leader`副本上, 这样可以保证较高的处理效率, 而`Follower`则会定期地到`Leader`上同步数据.
