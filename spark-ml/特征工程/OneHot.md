# OneHotEncoder

类：org.apache.spark.ml.feature.OneHotEncoder

方法：transform

```
    val encode = udf { label: Double =>
      if (label < size) {
        Vectors.sparse(size, Array(label.toInt), oneValue)
      } else {
        Vectors.sparse(size, emptyIndices, emptyValues)
      }
    }
```

核心代码是这行。其实spark都没做什么处理。因为返回的是一个稀疏向量，只需要输出label和值1。
``
        Vectors.sparse(size, Array(label.toInt), oneValue)

``


对比下python sklearn的OneHotEncoder。

```
    enc = OneHotEncoder()
    enc.fit([[0, 0, 3], [1, 1, 0], [0, 2, 1], [1, 0, 2]])
    print enc.transform([[0, 1, 1]]).toarray()
```

sklearn的OneHotEncoder是先fit,存在一个训练的过程，transform返回的是一个稠密向量
由于spark存储的是稀疏向量，只需要对DataFrame做简单的转换。可以不需要fit.
