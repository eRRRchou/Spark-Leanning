

## 1 Spark Streaming概念

* Spark Streaming是核心SparkAPI的扩展，实现实时数据流的可伸缩，高吞吐，容错处理。

* 数据的来源有Kafka，Flume和Tcp等等。
* 可以通过一些functions来处理，可以将处理后的数据推送到文件系统，数据库和实时仪表并在中间使用Spark机器学习和图计算。

![火花流](.gitimg/streaming-arch-1569418157901.png)

* 实际上是在内部将Dstream转成RDD在使用SparkAPI计算

  

![火花流](.gitimg\streaming-flow.png)

## 2 DStream

Spark Streaming提供的基本抽象。它表示连续的数据流，可以是从源接收的输入数据流，也可以是通过转换输入流生成的已处理数据流。DStream中的每个RDD都包含来自特定间隔的数据。

![火花流](.gitimg\streaming-dstream-1569418066663.png)

在DStream上执行的任何操作都转换为对基础RDD的操作。

![火花流](.gitimg\streaming-dstream-ops-1569418067367.png)

可以这么理解：



离线计算是执行了一个job，而实时计算执行的是多个小iob，每一个job都会有如下图所示的流程：

![WordCount-stage划分](.gitimg\WordCount-stage划分-1569418075240.jpg)

那将多个job合到一起，DStream就相当于多个RDD，就如同RDD包装partition一样，只不过是不同的时间段产生的。

如下图所示：

![SparkStreaming](.gitimg\SparkStreaming-1569418078719.jpg)

### 2.1 注意

* 在本地运行Spark Streaming程序时，请勿使用“ local”或“ local [1]”作为主URL。这两种方式均意味着仅一个线程将用于本地运行任务。如果使用的是基于接收器的输入DStream（例如tcp，Kafka，Flume等），则将使用单个线程来运行接收器，而不会处理接收到的数据。所以使用“ local [ *n* ]”作为主URL，其中*n* >要运行的接收者数。
* 不能再foreachRDD中初始化连接对象，因为RDD中的Partition可能存在不同的executor上面，但是连接对象是不能够序列化传输的，应该再foreachPartition中创建连接对象。
* 由基于窗口的操作生成的DStream会自动保存在内存中，而无需开发人员调用`persist()`。

### 2.2 UpdateStateByKey

| **updateStateByKey**（*func*） | 返回一个新的“状态” DStream，在该DStream中，通过在键的先前状态和键的新值上应用给定函数来更新每个键的状态。这可用于维护每个键的任意状态数据。 |
| ------------------------------ | ------------------------------------------------------------ |
| **transform(*func*)**          | **通过对源DStream的每个RDD应用RDD-to-RDD函数来返回新的DStream。这可用于在DStream上执行任意RDD操作。** |

updateStateByKey操作可以保持任意状态，同时不断用新信息更新它。要使用此功能，您将必须执行两个步骤。

1. 定义状态-状态可以是任意数据类型。
2. 定义状态更新功能-使用功能指定如何使用输入流中的先前状态和新值来更新状态。

在每个批次中，Spark都会对所有现有密钥应用状态更新功能，而不管它们是否在批次中具有新数据。如果更新函数返回None，则将删除键值对。

也就是会将之前统计的信息保存下来，下次统计会累加

代码如下所示：

```scala
  var updateFunction = (newValues: Seq[Int], runningCount: Option[Int]) => {
    val newCount = newValues.sum+runningCount.getOrElse(0)
    Some(newCount)
  }
  def main(args: Array[String]): Unit = {
     val conf = new SparkConf().setAppName("SparkStreamingWithAllNC").setMaster("local[2]")
     val sc = new SparkContext(conf)
     val ssc = new StreamingContext(sc,Seconds(2))
     sc.setCheckpointDir("D://checkpoint")
     val lines = ssc.socketTextStream("mini01",9999)
     val wordCounts = lines.flatMap(_.split(" +")).map((_,1)).updateStateByKey(updateFunction,new HashPartitioner(sc.defaultParallelism))
     wordCounts.print()
     ssc.start()
     ssc.awaitTermination()
  }
```

### 2.3 窗口操作

Spark Streaming还提供了*窗口计算*，可让您在数据的滑动窗口上应用转换。下图说明了此滑动窗口。

![火花流](.\.gitimg\streaming-dstream-window.png)

跟flink默认设置比较相似，就是每隔多少秒就去统计（多少秒之前的数据）

```
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10)) 
```

## 3 StreamingContext

要进行SparkStreaming程序，必须创建StreamingContext，该对象是所有Spark Streaming功能的主要入口点。

```
val conf = new SparkConf().setAppName(appName).setMaster(master)
val ssc = new StreamingContext(conf, Seconds(1))
```

定义上下文后，必须执行以下操作。

1. 通过创建输入DStream定义输入源。
2. 通过将转换和输出操作应用于DStream来定义流计算。
3. 开始接收数据并使用进行处理`streamingContext.start()`。
4. 等待使用停止处理（手动或由于任何错误）`streamingContext.awaitTermination()`。
5. 可以使用手动停止处理`streamingContext.stop()`。

## 4. 性能调优

* 减少批处理时间
* 考虑数据接收中的并行度
* 考虑数据处理中的并行度
* 数据序列化
* 任务启动开销
* 正确的批次间隔
* 内存调优

## 5.容错机制

* 高可用
* 分配足够内存
* 设置检查点
* 配置自动重新启动
* 预写日志
* 设置最大接收速率

## 6.Kafka

### 6.1 Kafka是什么

	kafka是一个生产-消费模型。
	Producer：生产者，只负责数据生产，生产者的代码可以集成到任务系统中。 数据的分发策略由producer决定，默认是defaultPartition  Utils.abs(key.hashCode) % numPartitions
	Broker：当前服务器上的Kafka进程，只管数据存储，不管是谁生产，不管是谁消费。在集群中每个broker都有一个唯一brokerid，不得重复。
	Topic:目标发送的目的地，这是一个逻辑上的概念，落到磁盘上是一个partition的目录。partition的目录中有多个segment组合(index,log)一个Topic对应多个partition[0,1,2,3]，一个partition对应多个segment组合。一个segment有默认的大小是1G。每个partition可以设置多个副本(replication-factor 1),会从所有的副本中选取一个leader出来。所有读写操作都是通过leader来进行的。
	特别强调：和mysql中主从有区别，mysql做主从是为了读写分离，在kafka中读写操作都是leader。
	ConsumerGroup：数据消费者组，ConsumerGroup可以有多个，每个ConsumerGroup消费的数据都是一样的。可以把多个consumer线程划分为一个组，组里面所有成员共同消费一个topic的数据，组员之间不能重复消费。

### 6.2 kafka生产数据时的分组策略

```
默认是defaultPartition  
Utils.abs(key.hashCode) % numPartitions
上文中的key是producer在发送数据时传入的，produer.send(KeyedMessage(topic,myPartitionKey,messageContent))
```

### 6.3 kafka如何保证数据的完全生产

```
ack机制：broker表示发来的数据已确认接收无误，表示数据已经保存到磁盘。
0：不等待broker返回确认消息
1：等待topic中某个partition leader保存成功的状态反馈
-1：等待topic中某个partition 所有副本都保存成功的状态反馈
```

### 4. broker如何保存数据

```
在理论环境下，broker按照顺序读写的机制，可以每秒保存600M的数据。
主要通过pagecache机制，尽可能的利用当前物理机器上的空闲内存来做缓存。
使用sendFile技术，不使用传统的IO将数据加载进内存，直接从内核中操作。
当前topic所属的broker，必定有一个该topic的partition，partition是一个磁盘目录。partition的目录中有多个segment组合(index,log)
```

### 5. partition如何分布在不同的broker上

	int i = 0
	list{kafka01,kafka02,kafka03}
	for(int i=0;i<5;i++){
		brIndex = i%broker;
		hostName = list.get(brIndex)
	}

### 6. consumerGroup的组员和partition之间如何做负载均衡

	算法：
		最好是一一对应，一个partition对应一个consumer。
		如果consumer的数量过多，必然有空闲的consumer。
		假如topic1,具有如下partitions: P0,P1,P2,P3
		加入group中,有如下consumer: C1,C2
		首先根据partition索引号对partitions排序: P0,P1,P2,P3
		根据consumer.id排序: C0,C1
		计算倍数: M = [P0,P1,P2,P3].size / [C0,C1].size,本例值M=2(向上取整)
		然后依次分配partitions: C0 = [P0,P1],C1=[P2,P3],即Ci = [P(i * M),P((i + 1) * M -1)]

### 7. 如何保证kafka消费者消费数据是全局有序的

```
伪命题
如果要全局有序的，必须保证生产有序，存储有序，消费有序。
由于生产可以做集群，存储可以分片，消费可以设置为一个consumerGroup，要保证全局有序，就需要保证每个环节都有序。
只有一个可能，就是一个生产者，一个partition，一个消费者。这种场景和大数据应用场景相悖。
```



## 7.Spark和Kafka集成指南

方法一：基于接收器

所有接收器一样，通过接收器从Kafka接收的数据存储在Spark执行器中，然后由Spark Streaming启动的作业将处理数据，此方法可能会在发生故障时丢失数据，需要另外启用预写日志，从而同步保存所有接收到的日志。将Kafka数据写入分布式文件系统（例如HDFS）上的预写日志中，以便在发生故障时可以恢复所有数据。

注意：

* Kafka中的主题分区与Spark Streaming中生成的RDD的分区不相关。

方法二：直连方法

定期向Kafka查询每个主题+分区中的最新偏移量，并相应地定义每个批次要处理的偏移量范围。

优点：

* Spark Streaming将创建与要使用的KafkaKafka分区一样多的RDD分区，所有这些分区都将从Kafka并行读取数据。

* 只要您有足够的Kafka保留时间，就可以从Kafka中再次读取数据。
* 确保一条信息只会被消费一次。

缺点：

缺点是它不会更新Zookeeper中的偏移量，因此基于Zookeeper的Kafka监视工具将不会显示进度。
