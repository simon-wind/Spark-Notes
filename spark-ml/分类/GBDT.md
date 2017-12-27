# GBDT
论文地址 ： http://en.wikipedia.org/wiki/Gradient_boosting

## 分类和回归
GBDT本质是回归树。如果label是0，1。模型是回归，需要先把label转换为-1，1

```
      case OldAlgo.Regression =>
        GradientBoostedTrees.boost(input, input, boostingStrategy, validate = false, seed)
      case OldAlgo.Classification =>
        // Map labels to -1, +1 so binary classification can be treated as regression.
        val remappedInput = input.map(x => new LabeledPoint((x.label * 2) - 1, x.features))
        GradientBoostedTrees.boost(remappedInput, remappedInput, boostingStrategy, validate = false,
          seed)
```

