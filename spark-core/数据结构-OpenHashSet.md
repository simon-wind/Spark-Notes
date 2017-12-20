# 背景

JDK的HashMap底层是用的数组，HashSet是基于HashMap（value为null）实现的。
为了适用大数据处理。Spark自己实现了一套高性能的HashMap和HashSet，据说比JDK自带的快一个数量级，缺点是只能添加元素不能删除。有意思的是跟JDK的实现相反。Spark的HashMap是用HashSet实现的。

# OpenHashSet实现原理


A simple, fast hash set optimized for non-null insertion-only use case, where keys are never
removed.

The underlying implementation uses Scala compiler's specialization to generate optimized
storage for two primitive types (Long and Int). It is much faster than Java's standard HashSet
while incurring much less memory overhead. This can serve as building blocks for higher level
data structures such as an optimized HashMap.

This OpenHashSet is designed to serve as building blocks for higher level data structures
such as an optimized hash map. Compared with standard hash set implementations, this class
provides its various callbacks interfaces (e.g. allocateFunc, moveFunc) and interfaces to
retrieve the position of a key in the underlying array.

It uses quadratic probing with a power-of-2 hash table size, which is guaranteed
to explore all spaces for each key (see http://en.wikipedia.org/wiki/Quadratic_probing).

 
基本原理:基于specialized,解决java泛型拆箱装箱带来的性能问题。

## 内部细节：

* 初始化的大小为64.扩展因子为0.7

```scala
  def this(initialCapacity: Int) = this(initialCapacity, 0.7)

  def this() = this(64)
```

* hash的key类型只能为Long和Int

```scala
  protected val hasher: Hasher[T] = {
    // It would've been more natural to write the following using pattern matching. But Scala 2.9.x
    // compiler has a bug when specialization is used together with this pattern matching, and
    // throws:
    // scala.tools.nsc.symtab.Types$TypeError: type mismatch;
    //  found   : scala.reflect.AnyValManifest[Long]
    //  required: scala.reflect.ClassTag[Int]
    //         at scala.tools.nsc.typechecker.Contexts$Context.error(Contexts.scala:298)
    //         at scala.tools.nsc.typechecker.Infer$Inferencer.error(Infer.scala:207)
    //         ...
    val mt = classTag[T]
    if (mt == ClassTag.Long) {
      (new LongHasher).asInstanceOf[Hasher[T]]
    } else if (mt == ClassTag.Int) {
      (new IntHasher).asInstanceOf[Hasher[T]]
    } else {
      new Hasher[T]
    }
  }
```


* 添加元素。两个步骤 1：添加元素   2：判断是否需要重新hash，需要则重新hash
```scala
  /**
   * Add an element to the set. If the set is over capacity after the insertion, grow the set
   * and rehash all elements.
   */
  def add(k: T) {
    addWithoutResize(k)
    rehashIfNeeded(k, grow, move)
  }
```

  添加元素这个方法的逻辑：
  用BitSet存储hash后的位置。_data数组存储具体数值。biset的index和_data数组的下标一一对应。
1. 如果hash值不存在。数组里面添加该key,biset添加index索引，size +1
2. 如果_data数组里面有该元素，return
3. 如果hash值存在，也就是有hash冲突了。采用线性探查（linear quadratic probing）。pos + 1,pos + 2 ...
回到第一步，直到找到一个没有值的pos.


```scala
  /**
   * Add an element to the set. This one differs from add in that it doesn't trigger rehashing.
   * The caller is responsible for calling rehashIfNeeded.
   *
   * Use (retval & POSITION_MASK) to get the actual position, and
   * (retval & NONEXISTENCE_MASK) == 0 for prior existence.
   *
   * @return The position where the key is placed, plus the highest order bit is set if the key
   *         does not exists previously.
   */
  def addWithoutResize(k: T): Int = {
    var pos = hashcode(hasher.hash(k)) & _mask
    var delta = 1
    while (true) {
      if (!_bitset.get(pos)) {
        // This is a new key.
        _data(pos) = k
        _bitset.set(pos)
        _size += 1
        return pos | NONEXISTENCE_MASK
      } else if (_data(pos) == k) {
        // Found an existing key.
        return pos
      } else {
        // quadratic probing with values increase by 1, 2, 3, ...
        pos = (pos + delta) & _mask
        delta += 1
      }
    }
    throw new RuntimeException("Should never reach here.")
  }
```

contains方法类似。用线性探测查找。由于biset的get的时间复杂度是O(1)，contains方法的效率应该还不错。
```scala
  /**
   * Return the position of the element in the underlying array, or INVALID_POS if it is not found.
   */
  def getPos(k: T): Int = {
    var pos = hashcode(hasher.hash(k)) & _mask
    var delta = 1
    while (true) {
      if (!_bitset.get(pos)) {
        return INVALID_POS
      } else if (k == _data(pos)) {
        return pos
      } else {
        // quadratic probing with values increase by 1, 2, 3, ...
        pos = (pos + delta) & _mask
        delta += 1
      }
    }
    throw new RuntimeException("Should never reach here.")
  }
```

# 总结

跟JDK的HashMap主要区别在于key->value元素的存储和Hash冲突的解决方法不同。
HashMap是用的数组。OpenHashMap是用的BitSet存储key，数据存储data。
HashMap的hash冲突解决是用的拉链法。OpenHashMap是用的线性探测

# 知识点
hash冲突解决方法。不同HashMap的实现原理