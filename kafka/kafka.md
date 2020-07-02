# 生产者

## 分区

每一个分区都自己维护一个offset（偏移量）

### 分区原因

- 方便在集群拓展，每个分区可以通过调整以适应它所在的机器，而一个topic可以有多个分区组成
- 可以提高并发，因为可以以分区为单位进行读写。

### 分区原则

- 指定分区，直接使用
- 未指定分区，但是指定key，可以通过key的value进行hash出一个分区
- 未指定key和partition，第一次调用时随机生成一个整数，后续每次调用都在这个整数上自增，将这个值和topic可用的分区总数求余得到partition值。

### 数据可靠性

#### ack

为了保证producer发送的数据，能可靠的发送到指定的topic，topic的能每个partition收到produce发送的数据后，都需要向produce发送ack（acknowledgement 确认收到）。如果produce收到ack，就会进行下一轮的发送，否则重新发送数据。

#### 同步策略

kafka选择的是全部同步完成后，才发送ack。

这样子带来的弊端是，网络延迟将会变高。

#### ISR

采用全部同步完成的方案后，会出现这样子的问题 ：

由于leader需要等到所有的follower都同步完了，才会返回ack信息。如果有一个follower出现故障，一直无法和leader同步，那么leader就一直等下去。

为了解决这个问题，leader维护了一个动态的ISR（in-sync replica set），意为和leader保持同步的follower集合。当ISR的follower完成数据同步后，就会向leader发送ack。如果leader长期未收到一个follower的ack信息，leader 就会将这个follower移除ISR。

于此同时，新的leader也是从ISR中选举的。

#### acks参数配置

- 0  producer不等待broker的ack，这种操作提供一个最低的延迟，broker一接受到还没有写入磁盘就返回，当broker故障时有可能丢失数据
- 1 produce等待broker的ack，partition的leader落盘成功就返回ack，follower同步成功之前，leader故障，也会丢失数据
- -1 || all producer等待broker的ack，partition的leader和follower全部落盘成功才返回ack。如果在folloer数据同步后，broker发送ack之前，leader出现故障，会造成数据重复。

#### HW

high watermark，指的是消费者能见到的最大的offset，ISR队列中最小的LEO

#### LEO

log end Offset,指的是每个副本最大的offset。

#### leader选举成功后要做的事情

选举成功后的leader首先会根据HW进行通知，让所有的follower根据自身情况进行对应的同步，简而言之，多删少补。

## API

属于异步发送，主要由两个线程负责-man和sender线程。main线程将消息发送给RecordAccumulator，Sender线程不断从RecordAccumulator中拉取消息到kafka broker中。

`send`方法后，还需要经过三个类，interceptors,Serializer,Partitioner

# 消费者

## 分区

### RoundRobin

将所有topic的分区全部打乱，按照一定的顺序排列，轮询发送给消费者组里的消费者。

### Range

- 首先，将分区按数字顺序排行序，消费者按消费者名称的字典序排好序 

- 然后，用分区总数除以消费者总数。如果能够除尽，则皆大欢喜，平均分配；若除不尽，则位于排序前面的消费者将多负责一个分区 
- 因为以上的特性，会出现负载不均衡的情况

## offset存储

0.9版本以前存在zk上，0.9版本及其以上版本，存储在本地custom_offset的topic里。
