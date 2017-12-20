# RDD的概念

## 源码里面对RDD的定义
 Internally, each RDD is characterized by five main properties:
*   A list of partitions
*   A function for computing each split
*   A list of dependencies on other RDDs
*   Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
*   Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)

翻译一下：

1. partition是spark计算的最小并行单位。不同的数据源partition生成方法略有不同。
2. 该RDD的计算函数
3. 父RDD
4. 分区类。决定每个key分在哪个区。默认为hash
5. 偏好的存储位置

举例：JdbcRDD是根据自定义的numPartitions和表的数据量生成partition
```scala
  override def getPartitions: Array[Partition] = {
    // bounds are inclusive, hence the + 1 here and - 1 on end
    val length = BigInt(1) + upperBound - lowerBound
    (0 until numPartitions).map { i =>
      val start = lowerBound + ((i * length) / numPartitions)
      val end = lowerBound + (((i + 1) * length) / numPartitions) - 1
      new JdbcPartition(i, start.toLong, end.toLong)
    }.toArray
  }
```

## 理解RDD的关键点
**1、不像一般的数据结构如，HashMap,ArrayBuffer。所有数据都在内存里面，是一个具体的东西。
RDD更抽象，RDD是惰性执行的，只有在需要它的时候，它才会调用compute函数，生成一个个partiiton。**

**2、partition分布在不同机器上，最小数据并行单位。**

