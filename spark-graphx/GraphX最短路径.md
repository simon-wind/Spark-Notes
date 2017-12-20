GraphX的工具里面有提供ShortestPaths计算最短路径。

采用的计算模型为：pregel

论文地址：

* pregel https://kowshik.github.io/JPregel/pregel_paper.pdf

* graphx http://www.istc-cc.cmu.edu/publications/papers/2013/grades-graphx_with_fonts.pdf


核心概念为超级步（super step）。SuperStep(i)的输出为SuperSep(i+1)的输入。

基本流程为:
1. 初始化，所有顶点状态为active
2. SuperSetp(k)  每个顶点合并上一个超级步的消息，计算要发给下个超级步的消息。每个超级步内的节点操作是并行的。
3. 每个顶点收到消息后置为激活状态，重复super step(k)操作,直至收敛。

对应到单源最短路径这个应用场景：停止迭代条件为找到最短路径或者达到最大迭代次数。

* pregel模型里面的超级步对应graphX的mapReduceTriplets。

* 每一次mapReduceTriplets即一次超级步的执行过程。


graphx论文：http://www.istc-cc.cmu.edu/publications/papers/2013/grades-graphx_with_fonts.pdf

Spark（所有节点的权重都为1）。单机版实现在算法导论上有详细的描述。需要用到松弛操作（relaxation）

graphx 实现的pregel计算模型如下，代码比较精简。

```
  private def makeMap(x: (VertexId, Int)*) = Map(x: _*)

  private def incrementMap(spmap: SPMap): SPMap = spmap.map { case (v, d) => v -> (d + 1) }

  private def addMaps(spmap1: SPMap, spmap2: SPMap): SPMap = {
    (spmap1.keySet ++ spmap2.keySet).map {
      k => k -> math.min(spmap1.getOrElse(k, Int.MaxValue), spmap2.getOrElse(k, Int.MaxValue))
    }(collection.breakOut) // more efficient alternative to [[collection.Traversable.toMap]]
  }
```
```
def run[VD, ED: ClassTag](graph: Graph[VD, ED], landmarks: Seq[VertexId]): Graph[SPMap, ED] = {
    val spGraph = graph.mapVertices { (vid, attr) =>
      if (landmarks.contains(vid)) makeMap(vid -> 0) else makeMap()
    }

    val initialMessage = makeMap()

    def vertexProgram(id: VertexId, attr: SPMap, msg: SPMap): SPMap = {
      addMaps(attr, msg)
    }

    def sendMessage(edge: EdgeTriplet[SPMap, _]): Iterator[(VertexId, SPMap)] = {
      val newAttr = incrementMap(edge.dstAttr)
      if (edge.srcAttr != addMaps(newAttr, edge.srcAttr)) Iterator((edge.srcId, newAttr))
      else Iterator.empty
    }

    Pregel(spGraph, initialMessage)(vertexProgram, sendMessage, addMaps)
  }
```


1. 初始化生成spGraph。初始顶点的状态设置为Map(1->0),其他所有vertex状态设置为Map()
2. 发送消息。只有有出路径的vertex能发送消息。
3. 合并消息。用addMaps路径长度加1.
4. 终止条件。没有可以发送的消息。

核心在于怎么判断没有可以发送的消息？所有的sendMessage函数函数返回空的Iterator

问题：高效计算的核心在mapReduceTriplets函数,map阶段sendmessege，reduce阶段addMaps，抽时间仔细看看。

```
      // Send new messages, skipping edges where neither side received a message. We must cache
      // messages so it can be materialized on the next line, allowing us to uncache the previous
      // iteration.
      messages = GraphXUtils.mapReduceTriplets(
        g, sendMsg, mergeMsg, Some((oldMessages, activeDirection)))
      // The call to count() materializes `messages` and the vertices of `g`. This hides oldMessages
      // (depended on by the vertices of g) and the vertices of prevG (depended on by oldMessages
      // and the vertices of g).
      messageCheckpointer.update(messages.asInstanceOf[RDD[(VertexId, A)]])
      activeMessages = messages.count()
```