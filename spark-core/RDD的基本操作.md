# RDD的基本操作

整体分为transformation和action算子。

## transformation算子
* map
* mapPartitions
* reduceByKey
* disintct
* filter
* ...

## action算子
* top()
* print()
* collect()
* foreachRDD()
* foreachPartition()
* ...

可以根据是否需要对外输出来粗略区分action和transformation算子。

## 要点
1. 只有action算子会触发Job。reduceByKey等导致数据集分布变化的操作会触发shuffle,shuffle会写磁盘，减慢计算速度。尽量少用。

2. 可以用mapPartition和foreachPartition等操作实现公共资源的复用，比如数据库链接，hbase table等等

3. spark ui可以清晰的看到stage,task,dag等等


谈谈spark.default.parallelism.
```
object Partitioner {
  /**
   * Choose a partitioner to use for a cogroup-like operation between a number of RDDs.
   *
   * If any of the RDDs already has a partitioner, choose that one.
   *
   * Otherwise, we use a default HashPartitioner. For the number of partitions, if
   * spark.default.parallelism is set, then we'll use the value from SparkContext
   * defaultParallelism, otherwise we'll use the max number of upstream partitions.
   *
   * Unless spark.default.parallelism is set, the number of partitions will be the
   * same as the number of partitions in the largest upstream RDD, as this should
   * be least likely to cause out-of-memory errors.
   *
   * We use two method parameters (rdd, others) to enforce callers passing at least 1 RDD.
   */
  def defaultPartitioner(rdd: RDD[_], others: RDD[_]*): Partitioner = {
    val rdds = (Seq(rdd) ++ others)
    val hasPartitioner = rdds.filter(_.partitioner.exists(_.numPartitions > 0))
    if (hasPartitioner.nonEmpty) {
      hasPartitioner.maxBy(_.partitions.length).partitioner.get
    } else {
      if (rdd.context.conf.contains("spark.default.parallelism")) {
        new HashPartitioner(rdd.context.defaultParallelism)
      } else {
        new HashPartitioner(rdds.map(_.partitions.length).max)
      }
    }
  }
}
```

采用默认的Partitioner后的partition个数。
如：reduceByKey，join等操作如果不自己定义numPartitions或Partition类，就会默认采用这个分区器。
