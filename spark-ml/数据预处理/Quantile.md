# Quantile

类似Pandas的quantile。统计学常用。

示例
```
    // $example on$
    val data = Array((0, 18.0), (1, 19.0), (2, 8.0), (3, 5.0), (4, 2.2))
    val df = spark.createDataFrame(data).toDF("id", "hour")
    // $example off$
    // Output of QuantileDiscretizer for such small datasets can depend on the number of
    // partitions. Here we force a single partition to ensure consistent results.
    // Note this is not necessary for normal use cases
        .repartition(1)

    // $example on$
    val discretizer = new QuantileDiscretizer()
      .setInputCol("hour")
      .setOutputCol("result")
      .setNumBuckets(3)

    val result = discretizer.fit(df).transform(df)
```
设定bucket大小，把vector拆分成一个个大小相等的区间。

跟单机版的pandas相比，如何在分布式环境下，快速决定每个区间的阈值。

用到了这个类：

org.apache.spark.sql.catalyst.util
论文地址： http://dx.doi.org/10.1145/375663.375670