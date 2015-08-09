# RDD 分区
## 分区
先回答第一个问题：RDD 内部，如何表示并行计算的一个计算单元。答案是使用__分区（Partition）__。

RDD 内部的数据集合在逻辑上（物理上则不一定）被划分成多个分片，这样的每一个分片我们将其称为分区，分区的个数会决定并行计算的粒度，而每一个分区数值的计算都是在一个单独的任务中进行，因此并行任务的个数，也是由 RDD（实际上是一个作业的末 RDD）分区的个数决定的，我会在 1.2 小节以及第二章中，具体说明分区与并行计算的关系。

在后文中，我会用下图所示图形来表示 RDD 以及 RDD 内部的分区，RDD 上方文字表示该 RDD 的类型，分区颜色为紫红色表示该 RDD 执行了持久化操作（`persist`、`Cache`、`checkpoint` 或者数据原本存储在存储介质当中），蓝色表示该 RDD 为普通 RDD。

![RDD and Partition](../media/images/section1/RDDAndPartition.png)

## 分区实现

分区的源码实现为 `Partition` 类。

``` scala
/**
 * A partition of an RDD.
 */
trait Partition extends Serializable {
  /**
   * Get the split's index within its parent RDD
   */
  def index: Int

  // A better default implementation of HashCode
  override def hashCode(): Int = index
}
```

RDD 只是数据集的抽象，分区内部并不会存储具体的数据。`Partition` 类内包含一个 `index` 成员，表示该分区在 RDD 内的编号，通过 RDD 编号 + 分区编号可以唯一确定该分区对应的块编号，利用底层数据存储层提供的接口，就能从存储介质（如：HDFS、Memory）中提取出分区对应的数据。

`RDD` 抽象类中定义了 `_partitions` 数组成员和 `partitions` 方法，`partitions` 方法提供给外部开发者调用，用于获取 RDD 的所有分区。`partitions` 方法会调用内部 `getPartitions` 接口，`RDD` 的子类需要自行实现 `getPartitions` 接口。

```scala
  @transient private var partitions_ : Array[Partition] = null
```

以 `MappedRDD` 类中的 `getPartitions` 方法为例。

``` scala
  override def getPartitions: Array[Partition] = firstParent[T].partitions
```

可以看到，`MappedRDD` 的分区实际上与父 RDD 的分区完全一致，这也符合我们对 `map` 转换操作的认知。
RDD 分区的一个分配原则是：尽可能使得分区的个数，等于集群核心数目。


创建操作中，程序开发者可以手动指定分区的个数，例如 `sc.parallelize (Array(1, 2, 3, 4, 5), 2)` 表示创建得到的 RDD 分区个数为 2，在没有指定分区个数的情况下，Spark 会根据集群部署模式，来确定一个分区个数默认值。

分别讨论 `parallelize` 和`textFile` 两种通过外部数据创建生成RDD的方法。


``` scala
  /** Distribute a local Scala collection to form an RDD.
   *
   * @note Parallelize acts lazily. If `seq` is a mutable collection and is
   * altered after the call to parallelize and before the first action on the
   * RDD, the resultant RDD will reflect the modified collection. Pass a copy of
   * the argument to avoid this.
   */
  def parallelize[T: ClassTag](seq: Seq[T], numSlices: Int = defaultParallelism): RDD[T] = {
    new ParallelCollectionRDD[T](this, seq, numSlices, Map[Int, Seq[String]]())
  }
```


对于本地模式，默认分区个数等于本地机器的 CPU 核心总数（或者是用户通过 `local[N]` 参数指定分配给 Apache Spark 的核心数目，见 `LocalBackend` 类），显然这样设置是合理的，因为把每个分区的计算任务交付给单个核心执行，能够保证最大的计算效率。

``` scala
  override def defaultParallelism() =
    scheduler.conf.getInt("spark.default.parallelism", totalCores)
```

若使用 Apache Mesos 作为集群的资源管理系统，默认分区个数等于 8（为什么？）（见 `MesosSchedulerBackend` 类）。

``` scala
  // TODO: query Mesos for number of cores
  override def defaultParallelism() = sc.conf.getInt("spark.default.parallelism", 8)
```

其他集群模式（Standalone 或者 Yarn），默认分区个数等于集群中所有核心数目的总和，或者 2，取两者中的较大值（见 `CoarseGrainedSchedulerBackend` 类）。

``` scala
  override def defaultParallelism(): Int = {
    conf.getInt("spark.default.parallelism", math.max(totalCoreCount.get(), 2))
  }
```


``` scala
  /** Default min number of partitions for Hadoop RDDs when not given by user */
  def defaultMinPartitions: Int = math.min(defaultParallelism, 2)
  
  /**
   * Read a text file from HDFS, a local file system (available on all nodes), or any
   * Hadoop-supported file system URI, and return it as an RDD of Strings.
   */
  def textFile(path: String, minPartitions: Int = defaultMinPartitions): RDD[String] = {
    hadoopFile(path, classOf[TextInputFormat], classOf[LongWritable], classOf[Text],
      minPartitions).map(pair => pair._2.toString).setName(path)
  }
```


## 分区内部记录个数
分区分配的另一个分配原则是：尽可能使同一 RDD 不同分区内的记录的数量一致。

对于转换操作得到的 RDD，如果是窄依赖，则分区记录数量依赖于父 RDD 中相同编号分区是如何进行数据分配的，如果是 Shuffle 依赖，则分区记录数量依赖于选择的分区器，哈希分区器无法保证数据被平均分配到各个分区，而范围分区器则能做到这一点。这部分内容我会在 1.6 小节中讨论。


``` scala
private object ParallelCollectionRDD {
  /**
   * Slice a collection into numSlices sub-collections. One extra thing we do here is to treat Range
   * collections specially, encoding the slices as other Ranges to minimize memory cost. This makes
   * it efficient to run Spark over RDDs representing large sets of numbers.
   */
  def slice[T: ClassTag](seq: Seq[T], numSlices: Int): Seq[Seq[T]] = {
    if (numSlices < 1) {
      throw new IllegalArgumentException("Positive number of slices required")
    }
    // Sequences need to be sliced at the same set of index positions for operations
    // like RDD.zip() to behave as expected
    def positions(length: Long, numSlices: Int): Iterator[(Int, Int)] = {
      (0 until numSlices).iterator.map(i => {
        val start = ((i * length) / numSlices).toInt
        val end = (((i + 1) * length) / numSlices).toInt
        (start, end)
      })
    }
    seq match {
      case r: Range.Inclusive => {
        val sign = if (r.step < 0) {
          -1
        } else {
          1
        }
        slice(new Range(
          r.start, r.end + sign, r.step).asInstanceOf[Seq[T]], numSlices)
      }
      case r: Range => {
        positions(r.length, numSlices).map({
          case (start, end) =>
            new Range(r.start + start * r.step, r.start + end * r.step, r.step)
        }).toSeq.asInstanceOf[Seq[Seq[T]]]
      }
      case nr: NumericRange[_] => {
        // For ranges of Long, Double, BigInteger, etc
        val slices = new ArrayBuffer[Seq[T]](numSlices)
        var r = nr
        for ((start, end) <- positions(nr.length, numSlices)) {
          val sliceSize = end - start
          slices += r.take(sliceSize).asInstanceOf[Seq[T]]
          r = r.drop(sliceSize)
        }
        slices
      }
      case _ => {
        val array = seq.toArray // To prevent O(n^2) operations for List etc
        positions(array.length, numSlices).map({
          case (start, end) =>
            array.slice(start, end).toSeq
        }).toSeq
      }
    }
  }
```

`textFile` 方法分区内数据的大小则是由 Hadoop API 接口 `FileInputFormat.getSplits` 方法决定（见 `HadoopRDD` 类），得到的每一个分片即为 RDD 的一个分区，分片内数据的大小会受文件大小、文件是否可分割、HDFS 中块大小等因素的影响，但总体而言会是比较均衡的分配。

``` scala
  override def getPartitions: Array[Partition] = {
    val jobConf = getJobConf()
    // add the credentials here as this can be called before SparkContext initialized
    SparkHadoopUtil.get.addCredentials(jobConf)
    val inputFormat = getInputFormat(jobConf)
    if (inputFormat.isInstanceOf[Configurable]) {
      inputFormat.asInstanceOf[Configurable].setConf(jobConf)
    }
    val inputSplits = inputFormat.getSplits(jobConf, minPartitions)
    val array = new Array[Partition](inputSplits.size)
    for (i <- 0 until inputSplits.size) {
      array(i) = new HadoopPartition(id, i, inputSplits(i))
    }
    array
  }
```
## 参考资料
1. [Spark Configuration - Spark 1.2.0 Documentation](https://spark.apache.org/docs/1.2.0/configuration.html)
2. [FileInputFormat (Apache Hadoop Main 2.6.0 API)](https://hadoop.apache.org/docs/r2.6.0/api/org/apache/hadoop/mapred/FileInputFormat.html)