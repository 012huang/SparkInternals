# RDD 依赖
## 依赖

RDD 的容错机制是通过记录更新来实现的，且记录的是粗粒度的转换操作。在外部，我们将记录的信息称为血统（Lineage）关系，而到了源码级别，Apache Spark 记录的则是 RDD 之间的__依赖（Dependency）__关系。在一次转换操作中，创建得到的新 RDD 称为子 RDD，提供数据的 RDD 称为父 RDD，父 RDD 可能会存在多个，我们把子 RDD 与父 RDD 之间的关系称为依赖关系，或者可以说是子 RDD 依赖于父 RDD。
依赖只保存父 RDD 信息，转换操作的其他信息，如提供给转换操作的参数（处理函数等），会在创建 RDD 时候，保存在新的 RDD 内。
依赖在 Apache Spark 源码中的对应实现是 `Dependency` 抽象类。

``` scala
/**
```

每个 `Dependency` 子类内部都会存储一个 `RDD` 对象，对应一个父 RDD，如果一次转换转换操作有多个父 RDD，就会对应产生多个 `Dependency` 对象，所有的 `Dependency` 对象存储在子 RDD 内部，通过遍历 RDD 内部的 `Dependency` 对象，就能获取该 RDD 所有依赖的父 RDD。

## 依赖分类
Apache Spark 将依赖进一步分为两类，分别是__窄依赖（Narrow Dependency）__和 __Shuffle 依赖（Shuffle Dependency，在部分文献中也被称为 Wide Dependency，即宽依赖）__。


![窄依赖](../media/images/section1/RDDDependencies/NarrowDependencies.png)
  override def getDependencies: Seq[Dependency[_]] = {
    rdds.map { rdd: RDD[_ <: Product2[K, _]] =>
      /* I: Partitioner 相同，则是 OneToOneDepdencency */
      if (rdd.partitioner == Some(part)) {
        logDebug("Adding one-to-one dependency with " + rdd)
        new OneToOneDependency(rdd)
      } else {
        /* I: Partitioner 不同，则是 ShuffleDependency */
        logDebug("Adding shuffle dependency with " + rdd)
        new ShuffleDependency[K, Any, CoGroupCombiner](rdd, part, serializer)
      }
    }
  }
```
窄依赖的实现在 `NarrowDependency` 抽象类中。
/**
```

`NarrowDependency` 要求子类实现 `getParent` 方法，用于获取一个分区数据来源于父 RDD 中的哪些分区（虽然要求返回 `Seq[Int]`，实际上却只有一个元素）。

窄依赖可进一步分类成一对一依赖和范围依赖，对应实现分别是 `OneToOneDependency` 类和`RangeDependency` 类。
一对一依赖表示子 RDD 分区的编号与父 RDD 分区的编号完全一致的情况，若两个 RDD 之间存在着一对一依赖，则子 RDD 的分区个数、分区内记录的个数都将继承自父 RDD。

``` scala
/**
```

### 范围依赖
范围依赖是依赖关系中的一个特例，只被用于表示 `UnionRDD` 与父 RDD 之间的依赖关系。相比一对一依赖，除了第一个父 RDD，其他父 RDD 和子 RDD 的分区编号不再一致，Apache Spark 统一将`unionRDD` 与父 RDD 之间（包含第一个 RDD）的关系都叫做范围以来。范围依赖的实现如下。

``` scala
/**
```

`RangeDepdencency` 类中 `getParents` 的一个示例如下图所示，对于 `UnionRDD` 中编号为 3 的分区，可以计算得到其数据来源于父 RDD 中编号为 1 的分区。

![范围依赖](../media/images/section1/RDDDependencies/RangeDependencyExample.png)
Shuffle 依赖的对应实现为 `ShuffleDependency` 类，其源码如下。
/**
```

`ShuffleDependency` 类中几个成员的作用如下：

- `rdd`：用于表示 Shuffle 依赖中，子 RDD 所依赖的父 RDD。

## 依赖与容错机制
介绍完依赖的类别和实现之后，回过头来，从分区的角度继续探究 Apache Spark 是如何通过依赖关系来实现容错机制的。下图给出了一张依赖关系图，`fileRDD` 经历了 `map`、`reduce` 以及`filter` 三次转换操作，得到了最终的 RDD，其中，`map`、`filter` 操作对应的依赖为窄依赖，`reduce` 操作对应的是 Shuffle 依赖。

![容错机制 - 1](../media/images/section1/RDDDependencies/FT_1.png)

假设最终 RDD 第一块分区内的数据因为某些原因丢失了，由于 RDD 内的每一个分区都会记录其对应的父 RDD 分区的信息，因此沿着依赖关系往回走，我们就能找到该分区数据最终来源于 `fileRDD` 的所有分区，再沿着依赖关系往后计算，即可得到丢失的分区数据。

![容错机制 - 1](../media/images/section1/RDDDependencies/FT_2.png)


在上一节中我们看到，在 RDD 中，可以通过__计算链（Computing Chain）__来计算某个 RDD 分区内的数据，我们也知道分区是并行计算的基本单位，这时候可能会有一种想法：能否把 RDD 每个分区内数据的计算当成一个并行任务，每个并行任务包含一个计算链，将一个计算链交付给一个 CPU 核心去执行，集群中的 CPU 核心一起把 RDD 内的所有分区计算出来。

答案是可以，这得益于 RDD 内部分区的数据依赖相互之间并不会干扰，而 Apache Spark 也是这么做的，但在实现过程中，仍有很多实际问题需要去考虑。进一步观察窄依赖、Shuffle 依赖在做并行计算时候的异同点。


![容错机制 - 1](../media/images/section1/RDDDependencies/NarrowDependency_PC.png)

再来观察 Shuffle 依赖的计算链，如图下方左侧的图中，既有窄依赖，又有 Shuffle 依赖，由于 Shuffle 依赖中，子 RDD 一个分区的数据依赖于父 RDD 内所有分区的数据，当我们想计算末 RDD 中一个分区的数据时，Shuffle 依赖处需要把父 RDD 所有分区的数据计算出来，如右侧的图所示 —— 而这些数据，在计算末 RDD 另外一个分区的数据时候，同样会被用到。

![容错机制 - 1](../media/images/section1/RDDDependencies/ShuffleDependency_PC.png)

如果我们做到计算链的并行计算的话，这就意味着，要么 Shuffle 依赖处父 RDD 的数据在每次需要使用的时候都重复计算一遍，要么想办法把父 RDD 数据保存起来，提供给其余分区的数据计算使用。