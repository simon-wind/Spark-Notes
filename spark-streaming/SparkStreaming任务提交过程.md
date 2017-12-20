有如下代码：思考任务提交流程是什么样的？

```
      /**
        * spark streaming Context
        */
      ssc = new StreamingContext(conf, Seconds(60))
      ssc.checkpoint("checkpoint")
      ...
      dstream.foreachRDD(rdd =>
                  rdd.repartition(1).foreachPartition(partition => {
```


1：首先在application提交后，根据StreamingContext设定的Duration（最多到毫秒级别，粒度太细了对spark没有意义，每次生成RDD都是重量级操作）,
在driver启动任务调度JobScheduler程序,JobScheduler调用JobGenerator。
JobGenerator里面封装了一个定时器，定时生成RDD。之后的处理过程跟批处理的流程是一样的:

生成RDD--->根据shuffle划分stage--->生成DAG--->执行

```
  private val timer = new RecurringTimer(clock, ssc.graph.batchDuration.milliseconds,
    longTime => eventLoop.post(GenerateJobs(new Time(longTime))), "JobGenerator")
```

2：JobSchedular在StreamingContext里面实例化。并且在StreamingContext.start函数里面启动JobGenerator的任务，然后把启动事件上报给listenerBus。
```
  private[streaming] val scheduler = new JobScheduler(this)

```



本质 : 根据StreamingContext里面的Duraion，由一个定时器定时生成RDD.
然后在这个RDD上执行transformation and action。其实就是在在批处理上封装了一层。


如果采用kafka高阶api，用receiver模式来接收数据的时候，
由于是定时消费一个时间段的所有数据放入executor内存生成RDD，
当业务高峰期程序处理不过来时会导致数据积压，节点会OOM.

所以一般需要添加backpressure参数。
该参数通过计算上一个RDD的处理时间来动态调整下一个RDD的处理速率，实现该功能的为RateEstimator。


```
        // in seconds, should be close to batchDuration
        val delaySinceUpdate = (time - latestTime).toDouble / 1000

        // in elements/second
        val processingRate = numElements.toDouble / processingDelay * 1000

        // In our system `error` is the difference between the desired rate and the measured rate
        // based on the latest batch information. We consider the desired rate to be latest rate,
        // which is what this estimator calculated for the previous batch.
        // in elements/second
        val error = latestRate - processingRate

        // The error integral, based on schedulingDelay as an indicator for accumulated errors.
        // A scheduling delay s corresponds to s * processingRate overflowing elements. Those
        // are elements that couldn't be processed in previous batches, leading to this delay.
        // In the following, we assume the processingRate didn't change too much.
        // From the number of overflowing elements we can calculate the rate at which they would be
        // processed by dividing it by the batch interval. This rate is our "historical" error,
        // or integral part, since if we subtracted this rate from the previous "calculated rate",
        // there wouldn't have been any overflowing elements, and the scheduling delay would have
        // been zero.
        // (in elements/second)
        val historicalError = schedulingDelay.toDouble * processingRate / batchIntervalMillis

        // in elements/(second ^ 2)
        val dError = (error - latestError) / delaySinceUpdate

        val newRate = (latestRate - proportional * error -
                                    integral * historicalError -
                                    derivative * dError).max(minRate)
```
