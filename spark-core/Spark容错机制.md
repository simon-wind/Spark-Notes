# Spark容错机制

只看Standalone模式,On Yarn类似。

## MASTER&WORKER
集群启动后，所有work调用sendRegisterMessageToMaster向Master注册。Worker和Master通信的模型采用Actor模型。以前采用akka框架，现在底层已经换成了netty。
Worker注册成功后，启动一个线程池定时向Master发送心跳消息。

worker的启动过程
1. 新建本地工作目录
2. shuffleService 启动（不知道是干嘛的，没用到过）
3. 启动work的web ui
4. 像master注册
5. 启动metricsSystem,metricsSystem负责各种指标的监控


```scala
  override def onStart() {
    assert(!registered)
    logInfo("Starting Spark worker %s:%d with %d cores, %s RAM".format(
      host, port, cores, Utils.megabytesToString(memory)))
    logInfo(s"Running Spark version ${org.apache.spark.SPARK_VERSION}")
    logInfo("Spark home: " + sparkHome)
    createWorkDir()
    shuffleService.startIfEnabled()
    webUi = new WorkerWebUI(this, workDir, webUiPort)
    webUi.bind()

    workerWebUiUrl = s"http://$publicAddress:${webUi.boundPort}"
    registerWithMaster()

    metricsSystem.registerSource(workerSource)
    metricsSystem.start()
    // Attach the worker metrics servlet handler to the web ui after the metrics system is started.
    metricsSystem.getServletHandlers.foreach(webUi.attachHandler)
  }
```
Work收到注册成功的回复后，启动一个线程池，通过心跳消息和Master保持联系：
```scala
forwordMessageScheduler.scheduleAtFixedRate(new Runnable {
          override def run(): Unit = Utils.tryLogNonFatalError {
            self.send(SendHeartbeat)
          }
        }, 0, HEARTBEAT_MILLIS, TimeUnit.MILLISECONDS)
```
时间间隔为60*1000/4 毫秒，即15秒 
```scala
  private val HEARTBEAT_MILLIS = conf.getLong("spark.worker.timeout", 60) * 1000 / 4

```


如果有个worker挂掉或者超时。Master的timeOutDeadWorkers方法会把该worker删除。

```scala
  /** Check for, and remove, any timed-out workers */
  private def timeOutDeadWorkers() {
    // Copy the workers into an array so we don't modify the hashset while iterating through it
    val currentTime = System.currentTimeMillis()
    val toRemove = workers.filter(_.lastHeartbeat < currentTime - WORKER_TIMEOUT_MS).toArray
    for (worker <- toRemove) {
      if (worker.state != WorkerState.DEAD) {
        logWarning("Removing %s because we got no heartbeat in %d seconds".format(
          worker.id, WORKER_TIMEOUT_MS / 1000))
        removeWorker(worker, s"Not receiving heartbeat for ${WORKER_TIMEOUT_MS / 1000} seconds")
      } else {
        if (worker.lastHeartbeat < currentTime - ((REAPER_ITERATIONS + 1) * WORKER_TIMEOUT_MS)) {
          workers -= worker // we've seen this DEAD worker in the UI, etc. for long enough; cull it
        }
      }
    }
  }
```

MASTER&WORKER只是守护进程。负责管理application的资源申请，application的新建和删除等等。
如果job提交给yarn,这一部分工作由yarn来完成。

// TODO 
DRIVER&Executor
