# Action算子执行过程

* 每个SparkContext实例化一个DAGScheduler，
* 每个DAGScheduler里面一个DAGSchedulerEventProcessLoop，DAGSchedulerEventProcessLoop这个类用来处理提交的Job。

* action算子调用runJob方法，把这个job的信息传给DAGSchedulerEventProcessLoop。
DAGSchedulerEventProcessLoop从调用者接收要处理的事件，启动一个后台线程，每次处理一个Job。
```
private val eventThread = new Thread(name) {
    setDaemon(true)

    override def run(): Unit = {
      try {
        while (!stopped.get) {
          val event = eventQueue.take()
          try {
            onReceive(event)
          } catch {
            case NonFatal(e) =>
              try {
                onError(e)
              } catch {
                case NonFatal(e) => logError("Unexpected error in " + name, e)
              }
          }
        }
      } catch {
        case ie: InterruptedException => // exit even if eventQueue is not empty
        case NonFatal(e) => logError("Unexpected error in " + name, e)
      }
    }

  }
```

* 该线程最终调用DAGScheduler的doOnReceive方法处理Submitted的事件。
* 通过submitStage函数递归的生成一个DAG.函数比较简单，如果有父Stage就递归的提交父Stage.
最后调用submitMissingTasks来执行每个Stage的task

```
  /** Submits stage, but first recursively submits any missing parents. */
  private def submitStage(stage: Stage) {
    val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
      logDebug("submitStage(" + stage + ")")
      if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
        val missing = getMissingParentStages(stage).sortBy(_.id)
        logDebug("missing: " + missing)
        if (missing.isEmpty) {
          logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
          submitMissingTasks(stage, jobId.get)
        } else {
          for (parent <- missing) {
            submitStage(parent)
          }
          waitingStages += stage
        }
      }
    } else {
      abortStage(stage, "No active job for stage " + stage.id, None)
    }
  }
```

