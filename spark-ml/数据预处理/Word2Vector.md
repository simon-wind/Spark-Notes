# Word2Vector
## 训练过程
### 训练参数
```
    val wordVectors = new feature.Word2Vec()
      .setLearningRate($(stepSize))
      .setMinCount($(minCount))
      .setNumIterations($(maxIter))
      .setNumPartitions($(numPartitions))
      .setSeed($(seed))
      .setVectorSize($(vectorSize))
      .setWindowSize($(windowSize))
      .setMaxSentenceLength($(maxSentenceLength))
      .fit(input)
```
* 学习速率LearningRate.默认为0.025

  setDefault(stepSize -> 0.025)

* 最小词频。默认为5.只有词频大于5的词语会添加到训练样本

  setDefault(minCount -> 5)

* 最大迭代次数。默认为1.

  setDefault(maxIter -> 1)

* 分区大小。默认是1

  setDefault(numPartitions -> 1)

* 随机数。默认值为该class Name的初始值

  setDefault(seed, this.getClass.getName.hashCode.toLong)

* 词向量大小。默认值为100
  setDefault(vectorSize -> 100)

* 窗口大小。Spark实现的是 skip-gram 模型。窗口大小为[-window,window]。window默认值为5

* 最大句子长度，默认值为1000.



### 训练过程

```
  def fit[S <: Iterable[String]](dataset: RDD[S]): Word2VecModel = {

    learnVocab(dataset)

    createBinaryTree()

    val sc = dataset.context

    val expTable = sc.broadcast(createExpTable())
    val bcVocab = sc.broadcast(vocab)
    val bcVocabHash = sc.broadcast(vocabHash)
    try {
      doFit(dataset, sc, expTable, bcVocab, bcVocabHash)
    } finally {
      expTable.destroy(blocking = false)
      bcVocab.destroy(blocking = false)
      bcVocabHash.destroy(blocking = false)
    }
```

* 学习词汇表
1. learnVocab函数里面通过reduceByKey统计每个词的词频。过滤掉词频过小的词汇，阈值为minCount。
结果返回给driver,再按照词频逆序排序。
2. 每个词对应一个id.其实就是一个简单的数组下标和值的一对一映射。
```
    vocab = words.map(w => (w, 1))
      .reduceByKey(_ + _)
      .filter(_._2 >= minCount)
      .map(x => VocabWord(
        x._1,
        x._2,
        new Array[Int](MAX_CODE_LENGTH),
        new Array[Int](MAX_CODE_LENGTH),
        0))
      .collect()
      .sortWith((a, b) => a.cn > b.cn)

    vocabSize = vocab.length
    ...
    var a = 0
    while (a < vocabSize) {
      vocabHash += vocab(a).word -> a
      trainWordsCount += vocab(a).cn
      a += 1
    }
```
* 生成哈夫曼树

* 训练
```
   val sentences: RDD[Array[Int]] = dataset.mapPartitions { sentenceIter =>
      // Each sentence will map to 0 or more Array[Int]
      sentenceIter.flatMap { sentence =>
        // Sentence of words, some of which map to a word index
        val wordIndexes = sentence.flatMap(bcVocabHash.value.get)
        // break wordIndexes into trunks of maxSentenceLength when has more
        wordIndexes.grouped(maxSentenceLength).map(_.toArray)
      }
    }
```
第一步：每个句子转换为Array[Int].大于maxSentenceLength的句子切分成多个句子。


```
    val newSentences = sentences.repartition(numPartitions).cache()

```

第二步：repartition

```
    val syn0Global =
      Array.fill[Float](vocabSize * vectorSize)((initRandom.nextFloat() - 0.5f) / vectorSize)
    val syn1Global = new Array[Float](vocabSize * vectorSize)
```
第三步：初始化词向量矩阵。用XORShiftRandom给矩阵赋随机值。

第四步：迭代训练


参考资料： http://www.cnblogs.com/pinard/p/7243513.html


