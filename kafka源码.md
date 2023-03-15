

概念

broker:

消费者

生产者

topic：一个topic可以分为多个partition

消费者组

partition：partition内部有序,一个partition可以分为多个segment，一个分区只能由一个组内消费者消费

leader：leader负责partition的读写

follower

segment：每个segment对应.index和.log两个文件

offset











# 1、producer发送数据

1. 确认数据要发送到的 topic 的 metadata 是可用的（如果该 partition 的 leader 存在则是可用的，如果开启权限时，client 有相应的权限），如果没有 topic 的 metadata 信息，就需要获取相应的 metadata；
2. 序列化 record 的 key 和 value；
3. 获取该 record 要发送到的 partition（可以指定，也可以根据算法计算）；
4. 向 accumulator 中追加 record 数据，数据会先进行缓存；
5. 如果追加完数据后，对应的 RecordBatch 已经达到了 batch.size 的大小（或者batch 的剩余空间不足以添加下一条 Record），则唤醒 `sender` 线程发送数据







KafkaProducer是线程安全的 多个线程间可以共享使用同一个KafkaProducer对象

