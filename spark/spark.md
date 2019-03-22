
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

### 持久化