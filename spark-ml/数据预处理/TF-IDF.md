# TF-IDF

## 示例代码

```
    val sentenceData = spark.createDataFrame(Seq(
      (0.0, "Hi I heard about Spark"),
      (0.0, "I wish Java could use case classes"),
      (1.0, "Logistic regression models are neat")
    )).toDF("label", "sentence")

    val tokenizer = new Tokenizer().setInputCol("sentence").setOutputCol("words")
    val wordsData = tokenizer.transform(sentenceData)

    wordsData.show()
    val hashingTF = new HashingTF()
      .setInputCol("words").setOutputCol("rawFeatures").setNumFeatures(20)

    val featurizedData = hashingTF.transform(wordsData)
    featurizedData.show()
    // alternatively, CountVectorizer can also be used to get term frequency vectors

    val idf = new IDF().setInputCol("rawFeatures").setOutputCol("features")
    val idfModel = idf.fit(featurizedData)

    val rescaledData = idfModel.transform(featurizedData)
    rescaledData.show()
```


逻辑：
1. tokenizer把文档切分成一个个单词
2. HashingTF对词进行hash。并计算文档的词频（TF：Term Frequency）。Hash算法可以选用Murmur3或者scala自带的hash.
3. 训练IDF模型.IDF的计算公式为。**总文件数目除以包含该词语的文件的数目，再将结果取对数。**
4. transform（）计算TF-IDF。
```
      while (j < n) {
        /*
         * If the term is not present in the minimum
         * number of documents, set IDF to 0. This
         * will cause multiplication in IDFModel to
         * set TF-IDF to 0.
         *
         * Since arrays are initialized to 0 by default,
         * we just omit changing those entries.
         */
        if (df(j) >= minDocFreq) {
          inv(j) = math.log((m + 1.0) / (df(j) + 1.0))
        }
        j += 1
      }
```


所以重点就在于分布式环境下怎么计算包含每个词的文档数？
```
  def fit(dataset: RDD[Vector]): IDFModel = {
    val idf = dataset.treeAggregate(new IDF.DocumentFrequencyAggregator(
          minDocFreq = minDocFreq))(
      seqOp = (df, v) => df.add(v),
      combOp = (df1, df2) => df1.merge(df2)
    ).idf()
    new IDFModel(idf)
  }
```
通过treeAggregate函数用mapreduce的方式计算得到IDF.



PS:Murmur3是一个高新能低碰撞的非加密Hash算法