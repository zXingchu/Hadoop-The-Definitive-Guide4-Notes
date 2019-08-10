
<!-- TOC -->

- [关于 Spark](#关于-spark)
    - [安装 Spark](#安装-spark)
    - [示例](#示例)
        - [Spark 应用、作业、阶段和任务](#spark-应用作业阶段和任务)
        - [Scala 独立应用](#scala-独立应用)
        - [Java 示例](#java-示例)
        - [Python 示例](#python-示例)
    - [弹性分布式数据集](#弹性分布式数据集)
        - [创建](#创建)
        - [转换和动作](#转换和动作)
            - [聚合转换](#聚合转换)
        - [持久化](#持久化)
            - [持久化级别](#持久化级别)
        - [序列化](#序列化)
            - [数据](#数据)
            - [函数](#函数)
    - [共享变量](#共享变量)
        - [广播变量](#广播变量)
        - [累加器](#累加器)
    - [剖析 Spark 作业运行机制](#剖析-spark-作业运行机制)
        - [作业提交](#作业提交)
        - [DAG 的构建](#dag-的构建)
        - [任务调度](#任务调度)
        - [任务执行](#任务执行)
    - [执行器和集群管理器](#执行器和集群管理器)
        - [运行在 YARN 上的Spark](#运行在-yarn-上的spark)
            - [YARN 客户端模式](#yarn-客户端模式)
            - [YARN集群模式](#yarn集群模式)
    - [延伸](#延伸)

<!-- /TOC -->

# 关于 Spark
* Apache Spark(http://spark.apache.org/)是用于大数据处理的集群计算框架
    * Spark与MapReduce在API和运行环境方面有许多相似之处
    * Spark与Hadoop紧密集成，可以在YARN上运行，并支持Hadoop文件格式及其存储后端(如HDFS)
* Spark最突出的表现在于它能将作业与作业之间产生的大规模的工作数据集存储在内存中
    * Spark在性能上超过了等效的MapReduce工作流，通常可高出1个数量级(MapReduce的数据集始终需要从磁盘上加载)
    * 从Spark处理模型中获益最大的两种应用类型分别为：迭代算法和交互式分析
* Spark提供三种语言的API：Scala、Java和Python
    * Spark为Scala和Python提供了REPL(read-eval-print-loop)的交互模式
* Apache Spark项目包括用于处理机器学习(MLlib)、图算法(GraphX)、流式计算(Spark Streaming)和SQL查询(Spark SQL)等模块
## 安装 Spark
* 从下载页面(http://spark.apache.org/downloads.html)下载一个稳定版本的Spark二进制发行包(与Hadoop发行版匹配)，解压缩
    ```
    % tar xzf spark-x.y.z-bin-distro.tgz
    ```
* 把Spark的二进制文件路径添加到路径
    ```
    % export SPARK_HOME=~/sw/spark-x.y.z-bin-distro
    % export PATH=$PATH:$SPARK_HOME/bin
    ```
## 示例
```
% spark-shell
Spark context available as sc.

scala> val lines = sc.textFile("input/ncdc/micro-tab/sample.txt)
lines: org.apache.spark.rdd.RDD[String] = MappedRDD[1] at textFile at <console>:12

scala> val records = lines.map(_.spilt("\t"))
records: org.apache.spark.rdd.RDD[Array[String]] = MappedRDD[2] at map at <console>:14

scala> val filtered = records.filter(rec => (rec(1) != "9999" && rec(2).matches("[01459]")))
filtered: org.apache.spark.rdd.RDD[Array[String]] = FilterRDD[3] at filter at <console>:16

scala> val tuples = filtered.map(rec => (rec(0).toInt, rec(1).toInt))
tuples: org.apache.spark.rdd.RDD[(Int, Int)] = MappedRDD[4] at map at <console>:18

scala> val maxTemps = tuples.reduceByKey((a, b) => Math.max(a, b))
maxTemps: org.apache.spark.rdd.RDD[(Int, Int)] = MapPartitionsRDD[7] at reduceByKey at <console>:21

scala> maxTemps.foreach(println(_))
(1950,22)
(1949,111)

scala> maxTemps.saveAsTextFile("output")
% cat output/part-*
(1950,22)
(1949,111)
```
* Spark-shell是添加了一些Spark功能的Scala REPL交互式解释器
* sc用于保存SparkContext实例。这是Spark的主要入口点
* lines变量引用的是一个弹性分布数据集(Resilient Distributed Dataset，简称RDD)
    * RDD是Spark最核心的概念，是在集群中跨多个机器分区存储的一个只读的对象集合
        * 弹性指Spark可以通过重新安排计算来自动重建丢失的分区

### Spark 应用、作业、阶段和任务
* Spark像MapReduce一样也有作业(job)的概念，且更通用
    * Spark作业是由任意的多阶段(stages)有向无环图(DAG)构成，其中每个阶段大致相当于MapReduce中的map阶段或reduce阶段
    * 这些阶段又被Spark运行环境分解成多个任务(task)，任务并行运行在分布式集群中的RDD分区上，就像MapReduce中的任务一样
* Spark作业始终运行在应用(application)上下文(用SparkContext实例来表示)中，它提供了RDD分组以及共享变量
* 一个应用可以串行或并行地运行多个作业，并为这些作业提供访问由同一应用的先前作业所缓存的RDD机制
    * 例如，spark-shell

### Scala 独立应用
```
import org.apache.spark.SparkContext._
import org.apache.spark.{SparkConf, SparkContext}

object MaxTemperature {
  def main(args: Array[String]) {
    val conf = new SparkConf().setAppName("Max Temperature")
    val sc = new SparkContext(conf)

    sc.textFile(args(0))
      .map(_.split("\t"))
      .filter(rec => (rec(1) != "9999" && rec(2).matches("[01459]")))
      .map(rec => (rec(0).toInt, rec(1).toInt))
      .reduceByKey((a, b) => Math.max(a, b))
      .saveAsTextFile(args(1))
  }
}
```
* 不是Spark定义的所有转换都可用于RDD类自身
    * 本例中，reduceByKey()实际上是在PairRDDFunctions类中定义的，可以使用下面的导入操作来让Scala将RDD[(Int, Int)]隐式转换为PairRDDFunctions
        * import org.apache.spark.SparkContext._
        * 此导入操作让Spark可以使用更多样化的隐式转换
```
% spark-submit --class MaxTemperature --master local \
  spark-examples.jar input/ncdc/micro-tab/sample.txt output
% cat output/part-*
(1950,22)
(1949,111)
```
* --class 指明Spark应用的类名

### Java 示例
```
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

public class MaxTemperatureSpark {
  
  public static void main(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.println("Usage: MaxTemperatureSpark <input path> <output path>");
      System.exit(-1);
    }

    SparkConf conf = new SparkConf();
    JavaSparkContext sc = new JavaSparkContext("local", "MaxTemperatureSpark", conf);
    JavaRDD<String> lines = sc.textFile(args[0]);
    JavaRDD<String[]> records = lines.map(new Function<String, String[]>() {
      @Override public String[] call(String s) {
        return s.split("\t");
      }
    });
    JavaRDD<String[]> filtered = records.filter(new Function<String[], Boolean>() {
      @Override public Boolean call(String[] rec) {
        return rec[1] != "9999" && rec[2].matches("[01459]");
      }
    });
    JavaPairRDD<Integer, Integer> tuples = filtered.mapToPair(
      new PairFunction<String[], Integer, Integer>() {
        @Override public Tuple2<Integer, Integer> call(String[] rec) {
          return new Tuple2<Integer, Integer>(
              Integer.parseInt(rec[0]), Integer.parseInt(rec[1]));
        }
      }
    );
    JavaPairRDD<Integer, Integer> maxTemps = tuples.reduceByKey(
      new Function2<Integer, Integer, Integer>() {
        @Override public Integer call(Integer i1, Integer i2) {
          return Math.max(i1, i2);
        }
      }
    );
    maxTemps.saveAsTextFile(args[1]);
  }
}
```
### Python 示例
```
from pyspark import SparkContext
import re, sys

sc = SparkContext("local", "Max Temperature")
sc.textFile(sys.argv[1]) \
  .map(lambda s: s.split("\t")) \
  .filter(lambda rec: (rec[1] != "9999" and re.match("[01459]", rec[2]))) \
  .map(lambda rec: (int(rec[0]), int(rec[1]))) \
  .reduceByKey(max) \
  .saveAsTextFile(sys.argv[2])
```

```
% spark-submit --master local \
  ch19-spark/src/main/python/MaxTemperature.py \
  input/ncdc/micro-tab/sample.txt output
```
* Spark通过一个名为PySpark的API来为Python提供语言支持
* Spark通过复刻(fork)Python子进程来运行用户的Python代码(在启动程序以及集群上运行用户任务的executor)，并且使用套接字连接两个进程，从而使父进程能够传递将要由Python代码处理的RDD分区数据

## 弹性分布式数据集
* 弹性分布式数据集(RDD)是所有Spark程序的核心

### 创建
* RDD的创建方法有三种：
    * 来自内存中的对象集合(也称为并行化一个集合)
    * 使用外部存储器(例如 HDFS)中的数据集
    * 对现有的RDD进行转换
* 第一种方法适用于少量的输入数据进行并行的CPU密集型计算。例如下面的代码对数字1到10运行独立计算：
    ```
    val params = sc.parallelize(1 to 10)
    val result = params.map(performExpensiveComputation)
    ```
    * performExpensiveComputation对输入的值并行运行，其并行度由spark.default.parallelism属性确定
        * spark.default.parallelism默认值取决于Spark作业的地点
            * 本地运行时，默认值就是该机器的内核数
            * 集群上运行时，为集群中所有executor节点的内核的总数
    * sc.parallelize(1 to 10, 10) 把并行度作为参数传递给parallelize()，可以覆盖特定计算的并行度
* 第二个方法是创建一个堆外数据集的引用
    ```
    val text: RDD[String] = sc.textFile(inputPath)
    sc.textFile(inputPath, 10)
    ```
    * inputPath可以是任何hadoop文件系统路径
    * Spark内部使用了旧的MapReduce API的TextInputFormat来读取文件，这意味着它的文件分割行为与Hadoop中的一致，因此在使用HDFS的情况下，每个HDFS块对应于一个Spark分区。此默认值可以通过传递第二个参数以请求特定的分割数量来更改
    ```
    val files: RDD[(String, String)] = sc.wholeTextFiles(inputPath)
    ```
    * 另一种变体，通过返回一个字符串对RDD来把文本文件作为一个完整的文件对待，第一个字符串为文件路径，第二个字符串是文件内容
        * 由于每个文件都要被加载到内存中，因此此方式只适合小文件
    ```
    sc.sequenceFile[IntWritable, Text](inputPath)
    sc.sequenceFile[Int, String](inputPath)
    ```
    * Spark也可以处理文本文件以外的其他格式。例如上述代码可用于读取顺序文件
        * 其中值得注意的是顺序文件键和值的Writable类型是如何指定的。对于普通的Writable类型，Spark可以将它们映射为等效的Java类型
    * 从任意Hadoop InputFormat格式创建RDD的方法有两种：
        * 对于需要路径输入的那些基于文件的格式可以使用hadoopFile()。新的MapReduce API，newApiHadoopFIle()
        * 对于不需要路径输入的格式则可以使用hadoopRDD()。新的MapReduce API，newApiHadoopRDD()
        ```
        val job = new Job()
        AvroJob.setInputKeySchema(job, WeatherRecord.getClassSchema)
        val data = sc.newAPIHadoopFile(inputPath,
            ClassOf[AvroKeyInputFormat[WeatherRecord]],
            ClassOf[AvroKey[WeatherRecord], ClassOf[NullWritable]],
            job.getConfiguration)
        ```

### 转换和动作
* Spark为RDD提供了两大类操作：转换(transformation)和动作(action)
    * 转换时从现有RDD生成新的RDD
    * 动作触发对RDD的计算并对计算结果执行某种操作，要么返回给用户，要么保存到外部存储器中
    * 动作的效果立竿见影，转换是惰性的，在对RDD执行一个动作之前都不会为该RDD的任何转换操作采取实际行动
* 判断一个操作是转换还是动作：观察其返回类型，返回类型是RDD为一个转换，否则为一个动作
* Spark库中包含了一组丰富的操作
    * 映射、分组、聚合、重新分区、采样、连接RDD以及把RDD作为集合来处理的各种转换
    * 将RDD物化为集合；对RDD进行统计数据的计算；从一个RDD中采样固定数量的元素；将RDD保存到外部存储器等各种动作

#### 聚合转换
```
val pairs: RDD[(String, Int)] = sc.parallelize(Array(("a", 3), ("a", 1), ("b", 7), ("a", 5)))
val sums: RDD[(String, Int)] = pairs.reduceByKey(_+_)
assert(sum.collect().toSet == Set(("a", 9), ("b", 7)))
val sums: RDD[(String, Int)] = pairs.foldByKey(0)(_+_)
assert(sum.collect().toSet == Set(("a", 9), ("b", 7)))

val sets: RDD[(String, HashSet[Int])] = pairs.aggregateByKey(new HashSet[Int])(_+=_, _++=_)
assert(sum.collect().toSet == Set(("a", Set(1, 3, 5)), ("b", Set(7))))
```
* 按键来为键-值对RDD进行聚合操作的三个主要转换函数分别是：reduceByKey()、foldByKey()和aggregateByKey()
    * 用于聚合给定键的值，并为每个键产生一个值。(对应的等效动作分别是reduce()、fold()和aggregate())
* reduceByKey()为键-值对中的值重复应用一个二进制函数，直至产生一个结果值
* foldByKey()与reduceByKey()类似，但必须提供一个零值(zero value)
* foldByKey()与reduceByKey()都不能改变聚合结果值的类型
* aggregateByKey()对PairRDD中相同的Key值进行聚合操作，在聚合过程中同样使用了一个中立的初始值
    * aggregateByKey返回值的类型不需要和RDD中value的类型一致
    * seqOp操作会聚合各分区中的元素，然后combOp操作把所有分区的聚合结果再次聚合，两个操作的初始值都是zeroValue
### 持久化
```
scala> tuples.cache()
res1: tuples.type = MappedRDD[4] at map at <console>:18
scala> tuples.reduceByKey((a, b) => Math.max(a, b).foreach(println(_)))
INFO BlockManagerInfo: Added rdd_4_0 in memory on 192.168.1.90:64640
(1950,22)
(1949,111)
scala> tuples.reduceByKey((a, b) => Math.min(a, b).foreach(println(_)))
INFO BlockManagerInfo: Found block rdd_4_0 locally
INFO BlockManagerInfo: Found block rdd_4_1 locally
(1949,78)
(1950,-11)
```
* 接示例的内容，上述命令把年份/温度对构成的中间数据集缓存到内存中
* 调用cache()并不会立即缓存RDD，相反，它用一个标志来对该RDD进行标记，以指示该RDD应当在Spark作业运行时被缓存
* 运行此作业，来自BlockManagerInfo的日志行表明RDD分区已作为作业运行的一部分保存在内存中
    * 日志显示RDD的编号为4(此编号在调用cache()方法后显示在控制台上)，它有两个分区，分别被标记为0和1
    * 运行另一个作业，看到从内存中加载该RDD
* 对于大规模作业来说，能够节省可观的时间
    * MapReduce在执行另一个计算时必须从磁盘中重新加载输入数据集，即使它可以使用中间数据集作为输出
    * Spark可以跨集群的内存中缓存数据集，使对数据集所做的任何计算都会非常快
    * 被缓存的RDD只能由同一应用的作业来读取
#### 持久化级别
*  调用cache()将会在executor的内存中持久化保存RDD的每个分区
   * 如果executor没有足够的内存来存储RDD分区，计算并不会失败，只不过是根据需要重新计算分区
   * 对于包含大量转换操作的复杂程序来说，重新计算的代价可能太高，因此Spark提供了不同级别的持久化行为
* 通过调用persist()并指定StorageLevel参数来做出选择
    * MEMORY_ONLY，默认的持久化级别，使用对象在内存中的常规表示方法
    * MEMORY_ONLY_SER，更紧凑的表示，把分区中的元素序列化为字节数组
        * 多了一份CPU开销，但如果生成的序列化RDD分区的大小适合被保存在内存中，常规方法不行，额外开销就是值得的
        * 还能减少垃圾回收的压力，因为每个RDD被存储为一个字节数据，而不是大量的对象
    *  MEMORY_AND_DOSK，如果数据集的大小不适合保存到内存中，就将其溢出到磁盘
    *  MEMORY_AND_DOSK_SER
### 序列化
* 默认情况下，RDD分区的序列化使用的是常规的Java序列化方法，但是无论从大小还是速度，使用Kryo序列化方法通常都是更好的选择
* 通过压缩序列化可以进一步节省空间(再次以牺牲CPU为代价)，其做法是把spaark.rdd.compress属性设置为true，并且可选的设置spark.io.compression.codec属性
* 在使用Spark时，要从两个方面来考虑序列化：数据序列化和函数序列化(或称为闭包函数)
#### 数据
* Kryo(https://github.com/EsotericSoftware/kryo)是一个高效的通用Java序列化库
* 使用Kryo序列化机制，需要在驱动程序的SparkConf中设置spark.seriallizer属性
    * conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
* Kyro不要求被序列化的类实现某个特定的接口
    * 因此，旧的纯Java对象也可以在RDD中使用
* 如果使用之前先在Kryo中对这些类进行注册，可以提高其性能
    * 因为Kryo需要写被序列化对象的类的引用(每个被写的对象都需要写一个引用)，如果已经注册了类，那么该引用就只是一个整数标识符，否则就是完整的类名
    * Spark已经注册了Scala类和其他一些框架类(比如Avro Generic类或Thrift类)
    * 在Kryo中注册类很简单，创建一个KryoRegistrator子类，然后重写registerClasses()方法。最后在driver程序中将spark.kryo.registrator属性设置为自定义的KryoRegistrator：
        ```
        class CustomKryoRegistrator extends KryoRegistrator {
          override def registerClasses(kyro: Kryo) {
            kyro.register(ClassOf[WeatherRecord])
          }
        }

        conf.set("spark.kryo.registrator", "KryoRegistrator")
        ```
#### 函数
* 通常函数的序列化会“谨守本分”：Scala中的函数都可以通过标准的Java序列化机制来序列化，这也是Spark用于向远程executor节点发送函数的手段
* 对Spark来说，即使在本地模式下运行，也需要序列化函数，假若引入了一个不可序列化的函数，在开发初期就应该会发现它
## 共享变量
```
val lookup = Map(1 -> "a", 2 -> "e", 3 -> "i", 4 -> "o", 5 -> "u")
val result = sc.parallelize(Array(2, 1, 3)).map(lookup(_))
assert(result.collect().toSet === Set("a", "e", "i"))
```
* Spark程序经常需要访问一些不属于RDD的数据。例如上述程在map()操作中用到了一张查找表(lookup table)
* 此段程序变量lookup作为闭包函数的一部分被序列化后传递给map()，但使用广播变量可以更高效地完成相同的工作
### 广播变量
* 广播变量(broadcast variable)在经过序列化后被发送给各个executor，然后缓存在那里，以便后期任务可以在需要时访问它
    * 常规变量是作为闭包函数的一部分被序列化的，因此在每个任务中都要通过网络被传输一次
    * 广播变量的作业类似于MapReduce中的分布式缓存，不同之处在于Spark将数据保存在内存中，内存耗尽才会溢出到磁盘上
```
val lookup: Broadcast[Map[Int, String]] = sc.broadcast(Map(1 -> "a", 2 -> "e", 3 -> "i", 4 -> "o", 5 -> "u"))
val result = sc.parallelize(Array(2, 1, 3)).map(lookup(_))
assert(result.collect().toSet === Set("a", "e", "i"))
```
* 通过SparkContext的broadcast()方法传递即将广播的变量创建一个广播变量。它返回Broadcast[T]，即对类型为T的变量的一个封装
    * 要想在RDD的map()操作中访问这些变量，需要对它们调用value
* 顾名思义，广播变量是单向传播的，即从driver到任务，因此一个广播变量是没有办法更新的，也不可能将更新传回driver
### 累加器
* 累加器(accumulator)是在任务中只能对它做加法的共享变量，类似于MapReduce中的计数器
* 当作业完成后，driver程序可以检索累加器的最终值
```
val count: Accumulator[Int] = sc.accumulator(0)
val result = sc.parallelize(Array(1, 2, 3))
  .map(i => { count += 1; i })
  .reduce((x, y) => x + y)
assert(count.value === 3)
assert(result === 6)
```
* 代码的第一行使用了SparkContext的accumulator()方法来创建一个累加器变量count
* Spark的累加器的类型可以是任意的数值类型，可以让累加器的结果类型与加数类型不同(参见SparkCOntext的accumulable()方法)，也能够累积可变集合中的值(通过accumulableCollection()方法)
## 剖析 Spark 作业运行机制
* 在最高层，Spark有两个独立的实体：driver和executor
    * driver负责托管应用(SparkContext)并为作业调度任务
    * executor专属于应用，它在应用运行期间运行，并执行该应用的任务
    * 通常，driver作为一个不由集群管理器(cluster manager)管理的客户端来运行，而executor则运行在集群的计算机上。并不总是这样
* 在下面的讨论中，假设应用的executor已经运行
![avatar](https://github.com/zXingchu/bigdata/blob/master/spark/Spark如何运行作业.png "Spark如何运行作业")
### 作业提交
* 当对RDD执行一个动作(比如count())时，会自动提交一个Spark作业。从内部看，它导致对SparkContext调用runJob()，然后将调用传递给作为driver的一部分运行的调度程序
* 调度程序由两部分组成：DAG调度程序和任务调度程序
    * DAG调度程序把作业分解为若干阶段，并由这些阶段构成一个DAG
    * 任务调度程序则负责把每个阶段的任务提交到集群
### DAG 的构建
* 在阶段中运行的任务有两种类型：shuffle map任务和result任务
    * shuffle map任务
        * 像MapReduce中shuffle的map端部分
        * 每个shuffle map任务在一个RDD分区上运行计算，并根据分区函数把输出写入一组新的分区中，以运行在后面的阶段中取用
        * shuffle map任务运行在除最终阶段之外的其他所有阶段中
    * result任务
        * result任务运行在最终阶段，并将结果返回给用户程序
        * 每个result任务在它自己的RDD分区上运行计算，然后把结果发送回driver，再由driver将每个分区的计算结果汇集成最终结果
* 最简单的Spark作业只有一个由result任务构成阶段
```
val hist: Map[Int, Long] = sc.textFile(inputPath)
  .map(word => (word.toLowerCase(), 1))
  .reduceByKey((a, b) => a + b)
  .map(_.swap)
  .countByKey()
```
![avatar](https://github.com/zXingchu/bigdata/blob/master/spark/用于计算词频统计分布图的Spark作业中的各个阶段以及RDD.png "用于计算词频统计分布图的Spark作业中的各个阶段以及RDD")
* 通常，每个阶段的RDD都要在DAG中显示，并且在DAG图中给出了这些RDD的类型以及创建它的操作。上图进行了一些简化，省略了Spark内部产生的一些中间RDD
* 由于reduceByKey()必须要有shuffle阶段，因此Spark的DAG调度程序将此作业分为两个阶段。如上图所示
    * 因为它是使用shuffle实现的，并且就像MapReduce一样，reduce函数一边在map端作为combiner运行，一边又在reduce端又作为reducer运行
    * 与MapReduce相似的另一个地方是，Spark的shuffle实现将其输出写入本地磁盘上的分区文件中(即使对内存中的RDD也一样)，并且这些文件将由下一个阶段的RDD读取
* DAG调度程序负责将一个阶段分解为若干任务以提交给任务调度程序
* DAG调度程序会为每个任务赋予一个位置偏好(placement preference)，以允许任务调度程序充分利用数据本地化(data locality)
    * 对于存储在HDFS上的输入RDD分区来说，它的任务的位置偏好就是托管了这些分区的数据块的datanode(称为node local)
    * 对于在内存中缓存的RDD分区，其任务的位置偏好则是那些保存RDD分区的executor(称为process local)
* 一旦DAG调度程序已构建了一个完整的多阶段DAG，它就将每个阶段的任务集合提交给任务调度程序
    * 子阶段只有父阶段成功完成后才能提交
### 任务调度
* 当任务集合被发送到任务调度程序后，任务调度程序用为该应用运行的executor的列表，在斟酌位置偏好的同时构建任务到executor的映射
* 接着，任务调度程序将任务分配给具有可用内核的executor(如果同一应用中的另一个作业正在运行，则有可能分配不完整)，并且在executor完成运行任务时继续分配更多的任务，直至任务集合全部完成
    * 默认情况下，每个任务到分派一个内核，也可以通过设置spark.task.cpus来更改
    * 任务调度程序为某个executor分配任务时，首先分配的是进程本地(process-local)任务，再分配节点本地(node-local)任务，人后分配机架本地(rack-local)任务，最后分配任意(非本地)任务或者推测任务(speculative task)，如果没有其他任务候选者的话
        * 推测任务是现有任务的复本，如果任务运行得比预期的慢，则调度程序可以将其作为备份来运行
* 如上图，被分配的任务通过调度程序后端启动。调度程序后端向executor后端发送远程启动任务的消息，以告知executor开始运行任务
    * Spark利用Akka(一个基于Actor的平台，http://akka.io/)来构建高度可扩展的事件驱动分布式应用，而不是Hadoop RPC进行远程调用
* 当任务成功完成或者失败时，executor都会向driver发送状态更新消息
    * 任务失败了，任务调度程序将在另一个executor上重新提交任务
    * 若启用了推测任务，还会为运行缓慢的任务启动推测任务
### 任务执行
* Executor以如下方式运行任务：
    * 首先，它确保任务的的JAR包和文件依赖关系都是最新的
        * executor在本地高速缓存中保留了先前任务已使用的所有依赖，因此只有在它们更新的情况下才会重新下载
    * 第二步，由于任务代码是以启动任务消息的一部分而发送的序列字节，因此需要反序列化任务代码(包括用户自己的函数)
    * 第三步，执行任务代码
        * 因为任务运行在与executor相同的JVM中，因此任务的启动没有进程开销
* 任务可以向driver返回执行结果。这些执行结果被序列化并发送到executor后端，然后以状态更新消息的形式返回driver
    * shuffle map任务返回的是一些可以让下一个阶段检索其输出分区的信息
    * result任务返回其运行的分区的结果值，driver将这些结果值收集起来，并把最终结果返回给用户的程序
## 执行器和集群管理器
* 负责管理executor生命周期的是集群管理器(cluster manager)，Spark提供了好多种不同特性的集群管理器
    * 本地模式
        * 独立模式的集群管理器是一个简单的分布式实现，它运行了一个master以及一个或多个worker
        * 当Spark应用启动时，master要求worker代表应用生成多个executor进程
        * 这种模式的主URL为spark://host:port
    * Mesos模式
        * Apache Mesos是一个通用的集群资源管理器，它允许根据组织策略在不同的应用之间细化资源共享
        * 默认情况下(细粒度模式)，每个Spark任务被当作是一个Mesos任务运行
            * 可以更有效地使用集群资源，但是以额外的进程启动开销为代价
        * 在粗粒度模式下，executor在进程中运行任务，因此在Spark应用运行期间的集群资源由executor进程来掌管
        * 这种模式的主URL为mesos://host:port
    * YARN模式
        * YARN是Hadoop中使用的资源管理器
        * 每个运行的Spark应用对应于一个YARN应用实例，每个executor在自己的YARN容器中运行
        * 这种模式的主URL为yarn-client或yarn-cluster
            * yarn-client 以客户端模式连接YARN集群。集群的位置可以在HADOOP_CONF_DIR 环境变量中找到
            * yarn-cluster 以集群模式连接YARN集群。集群的位置可以在HADOOP_CONF_DIR 环境变量中找到
### 运行在 YARN 上的Spark
* 在YARN上运行Spark提供了与其他Hadoop组件最紧密的集成，也是在已有Hadoop集群的情况下使用Spark的最简便的方法
* 在YARN上运行，Spark提供了两种部署方式：YARN客户端模式和YARN集群模式
    * YARN客户端模式
        * YARN客户端模式的driver在客户端运行
        * 对于具有任何交互式组件的程序都必须使用YARN客户端模式
        * 在构建Spark程序时，任何调试输出都是立即可见的
    * YARN集群模式
        * YARN集群模式的driver在YARN的application master集群上运行
        * 适用于生成作业(production job)，因为整个应用在集群上运行，这样做更易于保留日志文件以供稍后检查。如果application master出现故障，YARN还可以尝试重新运行该应用
#### YARN 客户端模式
![avatar](https://github.com/zXingchu/bigdata/blob/master/spark/在YARN客户端模式下Spark executor的启动流程.png "在YARN客户端模式下Spark executor的启动流程")
* (步骤1)在YARN客户端模式下，当driver构建新的SparkContext实例时就启动了与YARN之间的交互
* (步骤2)该Context向YARN资源管理器提交一个YARN应用
* (步骤3)YARN资源管理器则启动集群节点管理器上的YARN容器，并在其中运行一个名为SparkExecutorLauncher的application master
* (步骤4)ExecutorLauncher的工作是启动YARN容器中的executor，为此，ExecutorLauncher要向资源管理器请求资源
* (步骤5)然后启动ExecutorBackend进程作为分配给他的容器
* 每个executor在启动时都会连接回SparkContext，并注册自身
    * 这就向SparkContext提供了关于可运行任务的executor的数量及其位置的信息，这些信息被用在任务的位置偏好策略中
* 启动的executor的数量(默认为两个)、每个executor使用的内核数(默认为1)、内存量(默认为1024MB)在spark-shell、spark-submit或py-spark中设置
    ```
    % spark-shell --master yarn-client \
      --num-executors 4 \
      --executor-cores 1 \
      --executor-memory 2g
    ```
#### YARN集群模式
* 使用yarn-cluster，其他参数与客户端模式相同
![avatar](https://github.com/zXingchu/bigdata/blob/master/spark/在YARN集群模式下Spark executor的启动流程.png "在YARN客户端模式下Spark executor的启动流程")
* spark-submit客户端将会启动YARN应用，但是它不会运行任何代码。剩余过程与客户端模式相同，除了application master在为executor分配资源之前先启动driver程序(3b)以外
* 在这两种YARN模式下，executor都是在还没有任何本地数据位置信息之前先启动的，因此最终有可能会导致executor与存有作业所希望访问文件的datanode并不在一起
* Spark提供了一种方法，可以在YARN群集模式下运行时提供一些有关位置的提示，以提高数据本地性
* SparkContext构造函数可以使用第二个参数来传递一个优选位置。此优选位置是利用InputFormatInfo辅助类根据输入格式和路径计算得到的。例如，对于文本文件，我们使用的是TextInputFormat：
    ```
    val preferredLocations = InputFormatInfo.computePreferredLocations(
        Seq(new InputFormatInfo(new Configuration(), classOf[TextInputFormat],
        inputPath)))
    val sc = new SparkContext(conf, preferredLocations)
    ```
* 当向资源管理器请求分配时，application master需要用到这个优选位置(步骤4)
## 延伸
* Holden Karau&Andy Konwinski&Patrick Wendell&Matei Zaharia 《Learning Spark》 http://shop.oreilly.com/product/0636920028512.do
* Apache Spark(http://spark.apache.org/) 最新Spark发行版的更新文档