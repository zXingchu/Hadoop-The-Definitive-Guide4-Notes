<!-- TOC -->

- [关于 Hive](#关于-hive)
    - [安装 Hive](#安装-hive)
            - [Hive 的 shell 环境](#hive-的-shell-环境)
    - [示例](#示例)
    - [运行 Hive](#运行-hive)
        - [配置 Hive](#配置-hive)
            - [执行引擎](#执行引擎)
            - [日志记录](#日志记录)
        - [Hive 服务](#hive-服务)
            - [Hive 客户端](#hive-客户端)
        - [Metastore](#metastore)
    - [Hive 与传统数据库相比](#hive-与传统数据库相比)
        - [读时模式 VS 写时模式](#读时模式-vs-写时模式)
        - [更新、事务和索引](#更新事务和索引)
        - [其他SQL-on-Hadoop技术](#其他sql-on-hadoop技术)
    - [HiveQL](#hiveql)
        - [数据类型](#数据类型)
            - [原子类型](#原子类型)
            - [复杂类型](#复杂类型)
        - [操作与函数](#操作与函数)
            - [类型转换](#类型转换)
    - [表](#表)
        - [托管表和外部表](#托管表和外部表)
        - [分区和桶](#分区和桶)
            - [分区](#分区)
            - [桶](#桶)
        - [存储格式](#存储格式)
            - [默认存储格式：分隔的文本](#默认存储格式分隔的文本)
            - [二进制存储格式：顺序文件、Avro数据文件、Parquet文件、RCFile与ORCFile](#二进制存储格式顺序文件avro数据文件parquet文件rcfile与orcfile)
            - [使用定制的SerDe：RegexSerDe](#使用定制的serderegexserde)
            - [存储句柄](#存储句柄)
        - [导入数据](#导入数据)
            - [INSERT 语句](#insert-语句)
            - [多表插入](#多表插入)
            - [CREATE TABLE ... AS SELECT 语句](#create-table--as-select-语句)
        - [表的修改](#表的修改)
        - [表的丢弃](#表的丢弃)
    - [查询数据](#查询数据)
        - [排序和聚集](#排序和聚集)
        - [MapReduce 脚本](#mapreduce-脚本)
        - [连接](#连接)
            - [内连接](#内连接)
            - [外连接](#外连接)
            - [半连接](#半连接)
            - [map 连接](#map-连接)
        - [子查询](#子查询)
        - [视图](#视图)
    - [用户定义函数](#用户定义函数)
        - [UDTF](#udtf)
        - [写 UDF](#写-udf)
        - [写 UDAF](#写-udaf)
            - [更复杂的UDAF](#更复杂的udaf)
    - [延伸](#延伸)

<!-- /TOC -->
# 关于 Hive
* Hive是一个构建在Hadoop上的数据仓库框架
* Hive的设计目的是让精通SQL技能但Java编程技能相对较弱的分析师能够对存放在Hadoop的大规模数据集执行查询
    * 很多组织把它用作一个通用的、可伸缩的数据处理平台
    * 不适合用来开发复杂的机器学习算法。但它对很多分析任务有用

## 安装 Hive
* Hive一般在工作站上运行。它把SQL查询转换为一系列在Hadoop集群上运行的作业
* Hive把数据组织为表，通过这种方式为存储在HDFS上的数据赋予结构。
    * 元数据(如表模式)存储在metastore数据库中
* 刚开始使用Hive时，为了方便，可以让metastore运行在本地机器上
    * 此设置是默认设置
    * 此时，创建的Hive表的定义是在本地机器上，无法和其他用户共享这些定义
* 安装Hive的过程：
    * 首先必须在本地安装和集群上相同版本的Hadoop
        * 刚开始使用Hive时，可能会选择在本地以独立模式或伪分布式模式运行Hadoop
        * 只要确保hadoop可执行文件在相应路径中或设置HADOOP_HOME环境变量，就不必另外告诉Hive当前使用Hadoop版本
    * 下载Hive的一个发行版本(http://hive.apache.org/downloads.html)，把压缩包解压到工作站合适的位置
        ```
        % tar xzf apache-hive-x.y.z-bin.tar.gz
        ```
    * 把Hive放在路径下以便于访问
        ```
        % export HIVE_HOME=~/sw/apache-hive-x.y.z-bin
        % export PATH=$PATH:$HIVE_HOME/bin
        ```
    * 键入hive启动Hive的shell环境：
        ```
        % hive
        hive>
        ```

#### Hive 的 shell 环境
* Hive的shell环境是我们和hive交互，发出HiveQL命令的主要方式
    * HiveQL是Hive的查询语言。是SQL的一种“方言”。设计上很大程度上深受MySQL的影响
```
hive> SHOW TABLES;
```
* 列出Hive的表来检查Hive是否正常工作。命令必须以分号结束，告诉Hive立即执行该命令
* 和SQL类似，HiveQL一般是大小写不敏感的。制表符会自动补全Hive的关键字和函数
* 对于全新安装，此命令会话几秒钟来执行。因系统采用“延迟”(lazy)策略，所以直到此时才在机器上创建metastore数据库
    * metastore数据库把相关文件放在运行hive命令那个位置下的metastore_db目录中
* 可以以非交互式模式运行Hive的shell环境。使用-f选项可以运行指定文件中的命令
    ```
    % hive -f script.q
    ```
    * 对于较短的脚本，可用-e选项在行内嵌入命令。此时不需要表示结束的分号：
        ```
        % hive -e 'SELECT * FROM dummy'
        ```
    * 无论在交互式还是非交互式模式下，Hive都会把操作运行时的信息打印输出到标准错误输出(standard error)
        * 例如运行一个查询所花的时间。可以在启动程序的时候使用-S选项强制不显示这些信息，只输出查询结果
             ```
             % hive -S -e 'SELECT * FROM dummy'
             ```
    * 有一个较小的数据表用于测试查询是很有用的
        ```
        % echo 'x' > /tmp/dummy.txt
        % hive -e "CREATE TABLE dumm (value STRING); \
        LOAD DATA LOCAL INPATH '/tmp/dummy.txt' \
        OVERWRITE INTO TABLE dummy"
        ```
* 其他比较有用的Hive Shell的特性：
    * 使用!前缀来运行宿主操作系统的命令
    * 使用dfs命令来访问Hadoop文件系统

## 示例
* 如何用Hive查询气象数据
* 第一个步骤是把数据加载到Hive管理的存储，此处将让Hive把数据存储在本地文件系统
    * 和RDBMS一样，Hive把数据组织成表。使用CREATE TABLE语句为气象数据新建一个表：
        ```
        CREATE TABLE records (year STRING, temperature INT, quality INT)
        ROW FORMAT DELIMITED
          FIELDS TERMINATED BY '\t';
        ```
        * 第一行声明一个records表，包含三列：year，temperature，quality，指明每一列的数据类型
        * ROW FORMAT子句是HiveQL所特有的。此句所声明的是数据文件的每一行是由制表符分隔的文本
* 接下来，可以向Hive输入数据。此处只用一个很小的样本数据集：
     ```
     LOAD DATA LOCAL IMPATH 'input/ncdc/micro-tab/sample.txt'
     OVERWRITE INTO TABLE records;
     ```
     * 此命令告诉Hive把指定的本地文件放入其仓库目录中
     * 此操作并不解析文件或把它存储为内部数据库格式，因为Hive并不强制使用任何特定文件格式。文件以原样逐字存储
     * LOAD DATA语句中OVERWRITE关键字告诉Hive删除表对应目录中已有的所有文件
         * 无此关键字则把新文件加入目录，有同名文件则替换
* 此示例中，把Hive表存储在本地文件系统中(fs.default.name设为默认值file://)
    * 在Hive的仓库目录中，表存储为目录。仓库目录由选项hive.metastore.warehouse.dir控制
        * 选项hive.metastore.warehouse.dir默认值为/usr/hive/warehouse
    * 这样，records表的文件便可以在本地文件系统的/usr/hive/warehouse/records目录中找到：
        ```
        % ls /usr/hive/warehouse/record/
        sample.txt
        ```
        * 此示例只有一个文件sample.txt，但一般情况下，可以有多个文件，而且Hive会在查询表的时候读入所有这些文件
* 数据现在已经在Hive中，可以对它运行一个查询：
    ```
    hive> SELECT year, MAX(temperature)
        > FROM records
        > WHERE temperature != 9999 AND quality IN (0, 1, 4, 5, 9)
        > GROUP BY year;
    1949 111
    1950 22
    ```
## 运行 Hive

### 配置 Hive
* 和Hadoop类似，Hive用XML配置文件进行设置。配置文件为hivesite.xml，在Hive的conf目录下
    * 通过此文件可以设置每次运行Hive时希望希望Hive使用的选项
    * Hive的conf目录下还包括hive-default.xml(其中记录Hive使用的选项及其默认值)
    * 传递--config选项参数给hive命令，可以通过此方式重新定义Hive查找hive-site.xml文件的目录
        ```
        % hive --config /Users/tom/dev/hive-
        ```
        * 此选项指定的是包含配置文件的目录
        * 这对于有(对应于多个集群的)多个站点文件时很有用，方便在这些站点之间切换
        * 也可以设置HIVE_CONF_DIR环境变量来指定配置文件目录，效果相同
* hive-site.xml文件最适合存放详细的集群连接信息，因为可以使用Hadoop属性fs.defaultFS和yarn.resourcemanager.address来指定文件系统和资源管理器
    * 默认值使用本地文件系统和本地(正在运行的)“作业运行器”(job runner)
    * metastore的配置选项一般也能在hive-site.xml中找到
* Hive还运行向hive命令传递-hiveconf选项来为单个会话(per-session)设置属性
    ```
    % hive -hiveconf fs.defaultFS=hdfs://localhost \
      -hiveconf mapreduce.framework.name=yarn \
      -hiveconf yarn.resourcemanager.address=localhost:8032
    ```
* 还可以在一个会话中使用SET命令更改设置。对于某个特定的查询修改Hive设置非常有用
    * 例如，以下命令确保表的定义中都使用“桶”(bucket)
        ```
        hive> SET hive.enforce.bucketing=true;
        ```
    * 可以用只带属性名的SET命令查看任何属性的当前值：
        ```
        hive> SET hive.enforce.bucketing；
        hive.enforce.bucketing=true
        ```
    * 不带参数的SET命令会列出Hive所设置的所有属性。使用SET -v可以列出系统中的所有属性，包括默认值
* 如果准备让多个Hive用户共享一个Hadoop集群，则需要使Hive所用的目录对所有用户可写
    ```
    % hadoop fs -mkdir /tmp
    % hadoop fs -chmod a+w /tmp
    % hadoop fs -mkdir -p /user/hive/warehouse
    % hadoop fs -chmod a+w /user/hive/warehouse
    ```
    * 以上命令将创建目录并设置合适的权限
    * 如果所有用户在同一个用户组中，把仓库目录的权限设为g+w就够了
* 设置属性有一个优先级层次。在下面的列表中，越小的值表示优先级越高
    * (1) Hive SET命令
    * (2) 命令行-hiveconf选项
    * (3) hive-site.xml和Hadoop站点文件(core-site.xml、hdfs-site.xml、mapred-site.xml、yarn-site.xml)
    * (4) Hive默认值和Hadoop默认文件(core-default.xml、hdfs-default.xml、mapred-default.xml、yarn-default.xml)

#### 执行引擎
* Hive的原始设计是以MapReduce作为执行引擎(目前仍然是默认的执行引擎)
* 目前Hive的执行引擎还包括Apache Tez(http://tez.apache.org)
    * Hive对Spark的支持在开发中
    * Tez和Spark都是通用有向无环图(DAG)引擎，比MapReduce更灵活，性能更优越
* 具体使用哪种执行引擎由属性hive.execution.engine来控制，默认值为mr
* 基于每查询的执行引擎的切换操作非常简单。下述语句设置Tez为Hive的执行引擎(首先必须在Hadoop集群上安装Tez)：
    ```
    hive> SET hive.execution.engine=tez;
    ```
#### 日志记录
* 本地文件系统的${java.io.tmpdir}/${user.name}/hive.log中可以找到hive的错误日志
* 可以使用下述命令将日志目录指定到其他位置
    ```
    % hive -hiveconf hive.log.dir='/tmp/${user.name}'
    ```
* 日志的配置存放在conf/hive-log4j.properties中
    * 可编辑文件修改日志级别和其他日志相关设置
    * 更方便的办法是在会话中对日志配置进行设置，如
        ```
        % hive -hiveconf hive.root.logger=DEBUG,console
        ```
        * 将调试消息发送到控制台

### Hive 服务
* 可以在运行时使用--service选项指明使用的服务。键入hive --service help可以获得可用服务列表
* 一些最有用的服务：
    * cli
        * Hive的命令行接口(shell环境)。默认的服务
    * hiveservice2
        * 让Hive以提供Thrift服务的服务器形式运行，允许用不同语言编写的客户端进行访问
        * hiveservice2在支持认证和多用户并发方面比原始的hiveservice有很大改进
        * 使用Thrift、JDBC和ODBC连接器的客户端需要运行Hive服务器来和Hive进行通信
        * 通过设置hive.server2.thrift.port配置属性来指明服务器所监听的端口号(默认为10000)
    * beeline
        * 以嵌入方式工作的Hive命令行接口(类似于常规的CLI)，或使用JDBC连接到一个HiveServer2进程
    * hwi
        * Hive的Web接口
        * 在没有安装任何客户端软件的情况下这个简单的Web接口可以代替CLI
        * 另外，Hue是一个功能更全面的Hadoop Web接口，其中包括运行Hive查询和浏览Hive metastore的应用程序
    * jar
        * 与hadoop jar等价，这是运行类路径中包含Hadoop和Hive类Java应用程序的简便方法
    * metastore
        * 默认情况下，metastore和Hive服务运行在同一个进程里
        * 使用此服务可以让metastore作为一个单独的(远程)进程运行
        * 通过设置METASTORE_PORT环境变量(或使用-p命令行选项)可以指定服务器监听的端口号(默认为9083)
#### Hive 客户端
* 如果以服务器方式运行Hive(hive --service hiveserver2)，可以在应用程序中以不同机制连接到服务器
* Hive客户端和服务之间的联系：
    ![avatar](https://github.com/zXingchu/bigdata/blob/master/hive/Hive体系结构.png "Hive体系结构")
    * Thrift客户端
        * Hive服务器提供Thrift服务的运行，任何支持Thrift的编程语言都可与之交互
        * 有些第三方项目还提供Python和Rubt客户端。详情访问Hive维基页面(http://bit.ly/hive_server)
    * JDBC驱动
        * Hive提供了Type 4(纯Java)的JDBC驱动，定义在org.apache.hadoop.hive.jdbc.HiveDriver类中
        * 在以jdbc:hive2://host:port/dbname形式配置JDBC URI以后，java应用程序可以在指定的主机和端口连接到在另一个进程中运行的Hive服务器
        * 驱动使用Java的Thrift绑定来调用由Hive Thrift客户端实现的接口
        * 通过URI jdbc:hive2://，用JDBC内嵌模式来连接Hive
            * 此模式下Hive和发出调用的应用程序在同一个JVM中运行。这时不需要以独立服务器方式运行Hive
        * Beeline CLI 使用JDBC驱动与Hive通信
    * ODBC驱动
        * Hive的ODBC驱动允许支持ODBC协议的应用程序连接到Hive
        * Apache Hive的发布版本中没有ODBC驱动，有些厂商提供一个免费的ODBC驱动
            * 和JDBC驱动类似，ODBC驱动使用Thrift和Hive服务器进行通信
### Metastore
* metastore是Hive元数据的集中存放地
* metastore包括两部分：服务和后台数据的存储
![avatar](https://github.com/zXingchu/bigdata/blob/master/hive/Metastore的配置.png "Metastore的配置")
* 默认情况下，metastore服务和Hive服务运行在同一个JVM中，它包含一个内嵌的以本地磁盘作为存储的Derby数据库实例
    * 这称为内嵌metastore配置(embedded metastore configuration)，参见上图
    * 使用内嵌metastore是Hive入门最简单的方法
    * 一次只能为每个metastore打开一个Hive会话，因为每次只有一个内嵌Derby数据库可以访问某个磁盘上的数据库文件
        * 尝试启动第二个会话，试图连接metastore时，会得到错误信息
* 本地metastore配置(local metastore)：要支持多会话(以及多用户)，需要使用一个独立的数据库
    * metastore服务仍然和Hive服务运行在同一个进程中，但连接的却是在另一个进程中运行的数据库
    * 任何JDBC兼容的数据库都可以通过下表列出的javax.jdo.option.\*配置属性来供metastore使用
* 远程metastore配置(remote metastore)
    * 一个或多个metastore服务器和Hive服务运行在不同的进程内
    * 数据库层可以完全置于防火墙后，客户端不需要数据库凭据，提供了更好的可管理性和安全
    * 可以通过把hive.metastore.uris设为metastore服务器URI(逗号分隔)，把Hive服务设为使用远程metastore
        * metastore服务器URI的形式为thrift://host:port
        * 端口号对应于启动metastore服务器时所设定的METASTORE_PORT值
* 重要的metastore配置属性

| 属性名称 | 类型 | 默认值 | 描述 |
| ------ | ------ | ------ | ------ |
| hive.metastore.warehouse.dir | URI | /user/hive/warehouse | 相对于fs.default.name的目录，托管表就存储在这里 |
| hive.metastore.uris | 逗号分隔的URI | 未设定 | 如果未设置，则使用当前的metastore，否则连接到由URI列表指定要连接的远程metastore服务器。如果有多个远程服务器，则客户端以轮询(round robin)方式连接 |
| javax.jdo.option.ConnectionURL | URI | jdbc:derby:;databaseName=metastored b;create=true | metastore数据库的JDBC URL |
| javax.jdo.option.ConnectionDriverName | 字符串 | org.apache.derby.jdbc.EmbeddedDriver | JDBC驱动器的类名 |
| javax.jdo.option.ConnectionUserName | 字符串 | APP | JDBC用户名 |
| javax.jdo.option.ConnectionPassword | 字符串 | mine | JDBC密码 |
* 对于独立的metastore，MySQL是一种很受欢迎的选择
    * javax.jdo.option.ConnectionURL应设为jdbc:mysql://host/dbname? createDatabaseIfNotExist=true
    * javax.jdo.option.ConnectionDriverName应设为com.mysql.jdbc.Driver
    * MySQL的JDBC驱动的JAR文件(Connector/J)必须在Hive的类路径中，把此文件放入Hive的lib目录即可
## Hive 与传统数据库相比
* Hive很多方面和传统数据库类似，但起初对HDFS和MapReduce底层的依赖意味着它的体系结构有别于传统数据库，这些区别又影响着Hive所支持的特性
    * 随着时间的推移，局限性已逐渐消失，Hive越来越像传统数据库

### 读时模式 VS 写时模式
* 写时模式：
    * 在传统数据库里，表的模式是在数据加载时强制确定的。
    * 如果在加载时发现数据不符合模式，则拒绝加载。因为数据是在写入数据库时对照模式进行检查
* 读时模式
    * Hive对数据的验证不在加载数据时进行，而在查询时进行
* 需要在这两种方法之间进行权衡
    * 读时模式可以使数据加载非常迅速
        * 不需要读取数据来进行“解析”(parse)，再进行序列化并以数据库内部格式并入磁盘。数据加载操作仅仅是文件复制或移动
        * 更为灵活，例如针对不同的分析任务，同一个数据可能会有两个模式。Hive使用“外部表”(external table)时，此情况有可能发生
    * 写时模式有利于提升查询性能
        * 数据库可以对列进行索引，并对数据进行压缩
        * 但作为权衡，此时加载数据会花更多时间。此外在很多加载时模式未知的情况下，因为查询尚未确定，不能决定使用何种索引。Hive在这种情况下“长袖善舞”

### 更新、事务和索引
* 更新、事务和索引都是传统数据库最重要的特性
    * Hive没有考虑支持这些特性，因为Hive被设计为用MapReduce操作HDFS数据。在这样的环境下，“全表扫描”(full-table scan)是常态操作，而表更新则是通过把数据变换后放入新表实现的。对于在大规模数据集上运行的数据仓库应用，这一方式很见效

* 更新
    * 早前的Hive可利用INSERT INTO语句为表添加新的数据文件，以实现向现有表批量增加新行
    * 0.14.0版开始，Hive可对表做一些更细粒度的更新。可以使用INSERT INTO TABLE...VALUES插入少量通过SQL计算出来的值。还可对表中的行执行UPDATE和DELETE操作
    * HDFS不提供就地文件更新，因此，插入、更新和删除操作引起的一切变化都被保存在一个较小的增量文件中。由metastore在后台运行的MapReduce作业会定期将这些增量文件合并到“基表”(base table)文件中
* 上述功能只有在事务(由Hive0.13.0版引入)的背景环境下才能发挥作用，因此这些正在使用的表必须启用了事务，以保证对这些表进行读操作的查询可以看到表的一致性快照
* Hive0.7.0的发布版本还引入了表级(table-level)和分区级(partition-level)的锁
    * 有了锁可以防止一个进程删除正在被另一个进行读取的表
    * 锁由ZooKeeper透明管理，用户不必执行获得锁和释放锁的操作
    * 可通过SHOW LOCKS语句获取已经获得了哪些锁的信息
    * 默认情况下，并未启用锁的功能
* 目前Hive的索引分为两类：紧凑(compact)索引和位图(bitmap)索引
    * 索引的实现被设计为可插拔的，为其他目的而设计的索引会陆续出现
    * 紧凑索引存储每个值的HDFS块号，而不是存储文件内偏移量
        * 存储不会占用过多磁盘空间，且对于值被聚簇(clustered)存储于相近行的情况，索引仍然有效
    * 位图索引使用压缩的位集合(bitset)来高效存储具有某个特殊值的行
        * 适合于具有较少取值可能(low-cardinality)的列

### 其他SQL-on-Hadoop技术
* Cloudera Impala
    * 开源交互式SQL引擎，性能比基于MapReduce的Hive高一个数量级
    * 使用专用的守护进程，运行在集群中的每个数据节点上
* FaceBook Presto
* Apache Drill
* Spark SQL
    * 使用Spark作为其底层引擎，运行在Spark程序中嵌入SQL查询
* Apache Phoenix
    * 基于HBase的查询
    * 通过JDBC驱动实现SQL访问
## HiveQL
* Hive的SQL“方言”称为HiveQL，是SQL-92、MySQL和Oracle SQL语言的混合体
* 完整参考手册(http://bit.ly/languagemanual)
* SQL和HiveQL的概要比较

| 特性 | SQL | HiveQL |
| ------ | ------ | ------ |
| 更新 | UPDATE、INSERT、DELETE | UPDATE、INSERT、DELETE |
| 事务 | 支持 | 有限支持 |
| 索引 | 支持 | 支持 |
| 延迟 | 亚秒级 | 分钟级 |
| 数据类型 | 整数、浮点数、定点数、文本和二进制串、时间 | 布尔型、整数、浮点数、定点数、文本和二进制串、时间戳、数组、映射、结构 |
| 函数 | 数百个内置函数 | 数百个内置函数 |
| 多表插入 | 不支持 | 支持 |
| CREATE TABLE AS SELECT | SQL-92中不支持，但有些数据库支持 | 支持 |
| SELECT | SQL-92 | SQL-92。支持偏序的SORT BY。可限制返回行数量的LIMIT |
| 连接 | SQL-92支持或变相支持(FROM子句中列出连接表，在WHERE子句中列出连接条件) | 内连接、外连接、半连接、映射连接、交叉连接 |
| 子查询 | 在任何子句中支持“相关”的(correlated)或不相关的(noncorrelated) | 只能在FROM、WHERE或HAVING子句中(不支持非相关子查询) |
| 视图 | 可更新(是物化的或非物化的) | 只读(不支持物化视图) |
| 扩展点 | 用户定义函数 | 用户定义函数 |
|  | 存储过程 | MapReduce脚本 |

### 数据类型
* Hive的数据类型(列出的是在HiveQL中使用的形式而不是它们在表中序列化存储的格式)

| 类别 | 类型 | 描述 | 文字示例 |
| ------ | ------ | ------ | ------ |
| 基本数据类型 | BOOLEAN | true/false |  |
|  | TINYINT | 1字节(8位)有符号整数，从-128到127 | 1Y |
|  | SMALLINT | 2字节(16位)有符号整数，从-32768到32767 | 1S |
|  | INT | 4字节(32位)有符号整数，从-2147483648到2147483647 | 1 |
|  | BIGINT | 8字节(64位)有符号整数，从-2^64到2^64-1 | 1L |
|  | FLOAT | 4字节(32位)单精度浮点数 | 1.0 |
|  | DOUBLE | 8字节(64位)单精度浮点数 | 1.0 |
|  |  |  |  |
|  | DECIMAL | 任意精度有符号小数 | 1.0 |
|  | STRING | 无上限可变长度字符串 | 'a',"a" |
|  | VARCHAR | 可变长度字符串 | 'a',"a" |
|  | CHAR | 固定长度字符串 | 'a',"a" |
|  | BINARY | 字节数组 | 不支持 |
|  | TIMESTAMP | 精度到纳秒的时间戳 | 1325502245000，,2012-01-02 03:04:5.123456789 |
|  | DATE | 日期 | '2012-01-02' |
| 复杂数据类型 | ARRAY | 一组有序字段。字段的类型必须相同 | array(1,2) |
|  | MAP | 一组无序的键-值对。键的类型必须是原子的；值可以是任何类型。同一个映射的键的类型必须相同，值的类型也必须相同 | map('a',1,'b',2) |
|  | STRUCT | 一组命名的字段。字段的类型可以不同 | struct('a',1,1.0) named_struct('col1', 'a', 'col2', 1, 'col3', 1.0) |
|  | UNION | 值的数据类型可以是多个被定义的数据类型中的任意一个，这个值通过一个整数(零索引)来标记其为联合类型中的哪个数据类型 | create_union(1, 'a', 63) |
      * array()、map()、struct()三个函数都是Hive的内置函数

#### 原子类型
* 有些Hive的原子数据类型的命名受到MySQL数据类型名称的影响，但这些数据类型基本对应于Java中的类型
* DECIMAL类型用于表示任意精度的小数，类似于Java的BigDecimal类型，常常被用来表示货币值
    * DECIMAL的值以整数非标度值的形式保存
    * 精度(precision)指明非标度值的位数
    * 标度(scale)就是指小数点右侧的位数
    * DECIMAL(5,2)保存-999.99到999.99范围的值
    * 精度省略，默认值为10。精度的最大允许值为38，标度值不能超过精度值
* 存储文本的数据类型
    * STRING理论上最多能存储2GB的字符数
    * VARCHAR允许长度在1~65355之间
    * CHAR固定长度，必要则以空格填充尾部
        * 当CHAR值被用于字符串比较操作时，忽略尾部空格
* BINARY数据类型用于存储变长的二进制数据
* TIMESTAMP
    * Hive提供了在Hive时间戳、Unix时间戳、字符串之间进行转换的UDF
    * 并未封装时区信息。可以使用to_utc_timestamp和from_utc_timestamp函数进行时区转换

#### 复杂类型
* Hive有四种复杂数据类型：ARRAY、MAP、STRUCT和UNION
    * ARRAY和MAP与Java中的同名数据类型类似
* 复杂数据类型运行任意层次的嵌套。复杂数据类型声明必须用尖括号符号指明其中数据字段的类型
    ```
    CREATE TABLE complex (
        c1 ARRAY<INT>,
        c2 MAP<STRING, INT>,
        c3 STRUCT<a:STRING, b:INT, c:DOUBLE>,
        c4 UNIONTYPE<STRING, INT>
    );
    ```
    * 如下查询显示了上述每种类型的字段访问操作：
    ```
    hive> SELECT c1[0], c2['b'], c3.c, c4 FROM complex;
    1 2 1.0 {1:63}
    ```
### 操作与函数
* Hive提供了普通SQL操作符。这些操作符和MySQL的操作符一样
    * 关系操作符(例如等值判断：x='a';空值判断：x IS NULL;模式匹配：x LIKE 'a%')
    * 算术操作符(例如加分：x + 1)
    * 逻辑操作符(例如逻辑或：x OR y)
* Hive的内置函数分成几个大类：
    * 数学和统计函数
    * 字符串函数
    * 日期函数
    * 条件函数
    * 聚集函数
    * 处理XML(使用xpath函数)和JSON的函数
* 可以在Hive的shell环境中输入SHOW FUNCTIONS以获取函数列表
    * Hive函数参考手册(http://bit.ly/languagemanual_udf)
* 想了解特定函数，可以使用DESCRIBE命令。没有需要的内置函数，可以自己动手写
    ```
    hive> DESCRIBE FUNCTION length;
    ```
#### 类型转换
* 原子数据类型形成了一个Hive函数和操作符表达式进行隐式类型转换的层次
* Hive不会进行反向转换，会返回错误，除非使用CAST操作
* 隐式类型转换规则概述如下：
    * 任何数值类型都可以隐式地转换为一个范围更广的类型或者文本类型
    * 所有文本类型都可以隐式地转换为另一种文本类型
        * 文本类型都能隐式地转换为DOUBLE或DECIMAL
    * BOOLEAN类型不能转换为其他任何数据类型
    * TIMESTAMP和DATE可以被隐式地转换为文本类型
* 可以使用CAST操作显式进行数据类型转换
    * CAST('1' AS INT)将把字符串'1'转换成整数值1。强制类型转换失败，返回空值NULL
## 表
* Hive的表在逻辑上由存储的数据和描述表中数据形式的相关元数据组成
* Hive数据一般放在HDFS中，也可以放在其他任何Hadoop文件系统中，包括本地文件系统或S3
* Hive元数据存放在关系型数据库中，而不是HDFS中
### 托管表和外部表
* 在Hive中创建表时，默认情况下Hive负责管理数据
    * Hive把数据移入它的“仓库目录”(warehouse directory)
* 创建一个外部表(external table)
    * Hive到仓库目录以外的位置访问数据
* 托管表：
    * 加载数据到托管表时，Hive把数据移到仓库目录
        ```
        CREATE TABLE managed_table (dummy STRING);
        LOAD DATA INPATH '/user/tom/data.txt' INTO table managed_table;
        ```
        * 把文件hdfs://user/tom/data.txt移动到Hive的managed_table表的仓库目录中，即hdfs://user/hive/warehouse/managed_table
            * 只有源和目标文件在同一个文件系统中移动才会成功。使用LOCAL关键字，Hive会把本地文件系统的数据复制到仓库目录
        * 加载操作就是文件系统中的文件移动或文件重命名，执行速度很快
            * 即使是托管表，Hive也并不检查表目录中的文件是否符合为表所声明的模式
            * 只有在查询时才会知道有数据和模式不匹配
    * 丢弃一个表
        ```
        DROP TABLE managed_table;
        ```
        * 此表包括它的元数据和数据，会被一起删除
* 外部表：
    * 外部数据的位置需要在创建表的时候指明：
        ```
        CREATE EXTERNAL TABLE external_table (dummy STRING)
          LOCATION '/user/tom/external_table';
        LOAD DATA INPATH '/user/tom/data.txt' INTO TABLE external_table;
        ```
        *  使用EXTERNAL关键字
        *  定义时Hive甚至不会检查这一外部位置是否存在
    * 丢弃外部表时，Hive不会碰数据，只会删除元数据
* 多数情况下，这两种方式没有太大的区别。一个经验法则：
    *  所有处理都由Hive完成，应该使用托管表
    *  用Hive和其他工具来处理同一个数据集，应该使用外部表
    * 普遍的用法是把存放在HDFS(由其他进程创建)的初始数据集用作外部表进行使用，然后用Hive的变换功能把数据移到托管的Hive表
        *  反之，外部表(未必在HDFS中)可以用于从Hive导出数据供其他应用程序使用
              * 也可用 INSERT OVERWRITE DIRECTORY 把数据导出到Hadoop文件系统中
* 需要使用外部表的另一个原因是想为同一个数据集关联不同的模式

### 分区和桶
* Hive把表组织成分区(partition)
    * 根据分区列(partition column)的值对表进行粗略划分的机制
    * 使用分区可以加快数据分片(slice)的查询速度
* 表或分区可以进一步分为桶(bucket)
    * 为数据提供额外的结构以获得更高效的查询处理
        * 例如，根据用户ID来划分桶，可以在所有用户集合的随机样本上快速计算基于用户的查询
#### 分区
* 以分区的常用情况为例：考虑日志文件，其中每条记录包含一个时间戳，根据日期对它进行分区，同一天的记录被存放在同一个分区
    * 优点是对于限制到某个或某些特定日期的查询，处理可以变得非常高效，只需要扫描查询范围内分区中的文件
    * 使用分区并不会影响大范围查询的执行，仍然可以查询跨多个分区的整个数据集
    * 一个表可以以多个维度来进行分区。例如进一步根据国家对每个分区进行子分区(subpartition)，加速根据地理位置进行的查询
* 分区是在创建表的时候用 PARTITIONED BY 子句定义的。该子句需要定义列的列表
    * 在创建表后，可以使用 ALTER TABLE 语句来增加或移除分区
* 例如，对前面的假想的日志文件，可能要把表记录定义为由时间戳和日志行构成：
    ```
    CREATE TABLE logs (ts BIGINT, line STRING)
    PARTITIONED BY (dt STRING, country STRING);
    ```
    * 在把数据加载到分区表的时候，要显式指定分区值：
        ```
        LOAD DATA LOCAL INPATH 'input/hive/partitions/file1'
        INTO TABLE logs
        PARTITION (dt='2001-01-01', country='GB');
        ```
* 在文件系统级别，分区只是表目录下嵌套的子目录
    * 上述操作把更多文件加载到logs表以后，目录结构可能像下面这样：
        ```
            /user/hive/warehouse/logs
            |————dt=2001-01-01/
            |    |———country=GB/
            |    |   |——file1
            |    |   |__file2
            |    |___country=US/
            |        |__file3
            |____dt=2001-01-02/
                |———country=GB/
                |   |__file4
                |___country=US/
                    |__file5
                    |__file6
        ```
* 用 SHOW PARTITIONS 命令让Hive告诉我们表中有哪些分区：
    ```
    hive> SHOW PARTITIONS logs;
    dt=2001-01-01/country=GB
    dt=2001-01-01/country=US
    dt=2001-01-02/country=GB
    dt=2001-01-02/country=US
    ```
*  PARTITIONED BY 子句中的列定义是表中正式的列，称为分区列(partition column)，但数据文件并不包含这些列的值，它们源于目录名
*  可以在 SELECT 语句中以通常的方式使用分区列。Hive会对输入进行修剪：
    ```
    SELECT ts, dt, line
    FROM logs
    WHERE country='GB'
    ```
    * dt值Hive从目录中读取 
#### 桶
* 把表(或分区)组织成桶(bucket)有两个理由：
    * 获得更高的查询处理效率
        * 桶为表加上了额外的结构。Hive在处理有些查询时能够利用这个结构
        * 具体而言，连接两个在(包含连接列的)相同列上划分了桶的表。可以使用map端连接(map-side join)高效地实现
    * 使“取样”或者说“采样”(sampling)更高效
* 使用 CLUSTERED BY 子句来指定划分桶所用的列和要划分的桶的个数：
    ```
    CREATED TABLE bucketed_users (id INT, name STRING)
    CLUSTERED BY (id) INTO 4 BUCKETS;
    ```
    * 此处使用用户ID来确定如何划分桶(Hive对值进行哈希并将结果除以桶的个数取余数)
    * 对于map端连接的情况，首先两个表以相同方式划分桶，处理左边表内某个桶的mapper知道右边表内相匹配的行在对应的桶内，这样mapper只需要获取那个桶即可进行连接
        * 这一优化方法不一定要求两个表必须具有相同的桶的个数，两个表的桶个数是倍数关系也可以
* 桶中的数据可以根据一个或多个列另外进行排序
    * 对每个桶的连接变成了高效的归并排序(merge-sort)，因此可以进一步提升map端连接的效率
    * 以下语法声明一个表使其使用排序桶：
        ```
        CREATED TABLE bucketed_users (id INT, name STRING)
        CLUSTERED BY (id) SORTED BY (id ASC) INTO 4 BUCKETS;
        ```
* Hive并不检查数据文件中的桶是否和表定义中的桶一致(无论是对于同的数量或用于划分桶的列)
    * 两者不匹配，在查询时可能会碰到错误或未定义的结果
    * 建议让Hive来进行分桶的操作

* 一个没有划分桶的用户表：
    ```
    hive> SELECT * FROM users;
    0   Nat
    2   Joe
    3   Kay
    4   Ann
    ```
* 要向分桶后的表中填充成员，需要将hive.enforce.bucketing属性设置为true
    * Hive就知道用表定义中声明的数量来创建桶
    ```
    INSERT OVERWRITE TABLE bucketed_users
    SELECT * FROM users;
    ```
* 物理上，每个桶就是表(或分区)目录里的一个文件。它的文件名并不重要，桶n是按照字典序排列的第n个文件
    * 事实上，桶对应于MapReduce的输出文件分区：一个作业产生的桶(输出文件)和reduce任务个数相同
    * 运行如下命令查看刚才创建的bucketed_users表的布局
    ```
    hive> dfs -ls /user/hive/warehouse/bucketed_users;
    000000_0
    000001_0
    000002_0
    000003_0
    ```
    * 第一个桶里包括用户ID 0和4，因为一个INT的哈希值就是这个整数本身，在这里即除以桶数(4)以后的余数：
        ```
        hive> dfs -cat /user/hive/warehouse/bucketed_users/000000_0;
        0Nat
        4Ann
        ```
    * 用TABLESAMPLE子句对表进行取样，可以获得相同的结果。这个子句会将查询限定在表的一部分桶内，而不是使用整个表：
        ```
        // 返回约1/4的数据行
        hive> SELECT * FROM bucketed_users
            > TABLESAMPLE(BUCKET 1 OUT OF 4 ON id);
        4 Ann
        0 Nat
        // 返回约一半的数据行
        hive> SELECT * FROM bucketed_users
            > TABLESAMPLE(BUCKET 1 OUT OF 2 ON id);
        4 Ann
        0 Nat
        2 Joe
        hive> SELECT * FROM bucketed_users
            > TABLESAMPLE(BUCKET 1 OUT OF 4 ON rand());
        2 Joe
        ```
        * 桶的个数从1开始计数。查询只需要读取和TABLESAMPLE子句匹配的桶，所以取样分桶表示非常高效的操作
        * 如果使用rand()函数对没有划分成桶的表进行取样，即使只需要读取很小一部分样本，也要扫描整个输入数据集
### 存储格式
* Hive从两个维度对表的存储进行管理：行格式(row format)和文件格式(file format)
* 行格式指行和一行中的字段如何存储
    * 按照Hive的术语，行格式的定义由SerDe(序列化和反序列化工具 Serializer-Deserializer)定义
    * 当作为反序列化工具进行使用时(查询表时)，SerDe将把文件中字节形式的数据行反序列化为Hive内部操作数据行时所使用的对象形式
    * 使用序列化工具时(执行INSERT或CTAS)，表的SerDe会把Hive的数据行内部表示形式序列化成字节形式并写到输出文件中
* 文件格式指一行中字段容器的格式
    * 最简单的格式是纯文本文件
    * 也可以使用面向行的和面向列的二进制格式
#### 默认存储格式：分隔的文本
* 创建表时没有用ROW FORMAT或STORED AS子句，那么Hive所使用的默认格式是分隔的文本，每行存储一个数据行
* 分隔符
    * 默认的行内分隔符不是制表符，而是ASCII控制码集合中的Control-A(ASCII码为1)
        * 选择Control-A作为分隔符因为和制表符相比，出现在字段文本中的可能性比较小
        * 在Hive中无法对分隔符进行转义，挑选一个不会再数据字段中用到的字符作为分隔符非常重要
    * 集合类元素的默认分隔符为字符Control-B
        * 用于分隔ARRAY或STRUCT或MAP的键-值对中的元素
    * 默认的映射键(map key)分隔符为字符Control-C
        * 用于分隔MAP的键和值
    * 表中各行用换行符分隔
    ```
    CREATE TABLE ...;
    ```
    等价于
    ```
    CREATE TABLE ...
    ROW FORMAT DELIMITED
        FIELDS TERMINATED BY '\001'
        MAP KEYS TERMINATED BY '\002'
        LINES KEYS TERMINATED BY '\n'
    STORED AS TEXTFILE;
    ```
    * 可以使用八进制形式来表示分隔符，例如001表示Control-A
    * 对于嵌套数据类型，嵌套的层次(level)决定了使用哪种分隔符
        * 例如对于数组的数组，外层的数组的分隔符如前所述Control-B字符，内层数组则使用分隔符列表中的下一项(Control-C字符)作为分隔符
        * 不确定Hive使用哪个字符作为某个嵌套结构的分隔符，可以运行以下命令：
            ```
            CREATE TABLE nested
            AS
            SELECT array(array(1, 2), array(3, 4))
            FROM dummy;
            ```
        * 然后再使用hexdump或类似命令来查看输出文件的分隔符
        * 
    * Hive支持8级分隔符，分别对应于ASCII编码的1,2,......,8
        * 只能重载其中的前三个
* 分隔格式处理：Hive在内部使用一个名为LazySimpleSerDe的SerDe来处理这种分隔格式
    * 面向行的MapReduce文本输入和输出格式也是用此处理
    * 此SerDE对字段的序列化是延迟处理的，只有在访问字段时才进行反序列化
    * 由于文本以冗长的形式存放，此种存储格式并不紧凑
    * 使用其他工具(MapReduce程序或Streaming)来处理这样的格式非常容易
        * 但还可以选择一些更紧凑和高效的二进制SerDe

#### 二进制存储格式：顺序文件、Avro数据文件、Parquet文件、RCFile与ORCFile
* 通过 CREATE TABLE 语句中的 STORED AS 子句做相应声明
    * 不需要指定 ROW FORMAT，因为其格式由底层的二进制文件格式来控制
* 二进制格式可划分为两大类：面向行的格式和面向列的格式
    * 面向列的存储格式比较适用于只访问一小部分列的查询
    * 面向行的存储格式适合同时处理一行中很多列的情况
* Hive本身支持两种面向行的格式：Avro数据文件和顺序文件
    * 都是通用的可分割、可压缩的格式
    * Avro还支持模式演化以及多种编程语言的绑定
    ```
    SET hive.exec.compress.output=true;
    SET avro.output.codec=snappy;
    CREATE TABLE ... STORED AS AVRO;
    ```
    * 使用上述语句可以将表存储我Avro格式。设置响应属性可支持表压缩
    * 顺序文件 STORED AS SEQUENCEFILE
* Hive本身可支持的面向列的格式包括：Parquet、RCFile和ORCFile
    ```
    CREATE TABLE users_parquet STORED AS PARQUET
    AS
    SELECT * FROM users;
    ```
#### 使用定制的SerDe：RegexSerDe
* 使用定制的SerDe来加载数据(定制SerDe：采用正则表达式从一个文本文件中读取定长的观测站元数据)
    ```
    CREATE TABLE stations (usaf STRING, wban STRING, name STRING)
    ROW FORMAT SERDE 'org.apache.hadoop.hive.contrib.serde2.RegexSerDe'
    WITH SERDEPROPERTIES (
        "input.regex" = "(\\d{6}) (\\d{5}) (.{29}) .*"
    );
    ```
    * 用 SERDE 关键字和实现SerDe的Java类的完整类名，来指明使用哪个SerDE
    * SerDe可以用 WITH SERDEPROPERTIES 子句来设置额外的属性。此处设置RegexSerDe特有的input.regex属性
        * input.regex是在反序列化期间将要使用的正则表达式模式，用来将数据行(row)中的部分文本转化为列的集合
            * 正则表达式匹配时使用Java的正则表达式语法(http://bit.ly/java_regex)
            * 通过识别一组一组的括号来确定列(称为捕获组，即capturing group)
                * 在左括号后加问号 ? 表示“非捕获组”(noncapturing group)例如(?ab+)。有多种表示非捕获组的构造结构
                * 此示例有三个捕获组：usaf(六位数的标识符)、wban(五位数的标识符)以及name(29个字符的定长列)

```
LOAD DATA LOCAL INPATH "input/ncdc/metadata/stations-fixed-width.txt INTO
TABLE stations;
// 简单查询，反序列化会调用SerDe
hive> SELECT * FROM stations LIMIT 4;
10000 99999 BOGUS NORWAY
010003 99999 BOGUS NORWAY
010010 99999 JAN MAYEN
010013 99999 ROST
```
* 上例RegexSerDe能够帮助Hive获取数据，但是它的效率很低，因此一般不用于通用存储格式，应当考虑把数据复制为二进制存储格式

#### 存储句柄
* 存储句柄(Storage handler)用于Hive自身无法访问的存储系统，比如HBase
    * 存储句柄使用 STORED BY 子句来指定。代替了 ROW FORMAT 和 STORED AS 子句
    * 有关HBase集成的详细信息可以参考Hive的英文维基页面(http://bit.ly/hbase_int)

### 导入数据
* LOAD DATA
* INSERT
* CTAS，CREATE TABLE ... AS SELECT
* 关系型数据库导入Hive：Sqoop
#### INSERT 语句
```
INSERT OVERWRITE TABLE target
SELECT col1, col2
  FROM source;
```
* 简单示例
* OVERWRITE 意味着目标表内容会被替换
* INSERT INTO TABLE target
```
INSERT OVERWRITE TABLE target
PARTITION (dt='2001-01-01')
SELECT col1, col2
  FROM source;
```
* 分区的表，可以使用 PARTITION 子句来指明数据要插入哪个分区
```
INSERT OVERWRITE TABLE target
PARTITION (dt)
SELECT col1, col2, dt
  FROM source;
```
* 可以在 SELECT 语句中通过使用分区值来动态指明分区
    * 这种方法称为动态插入(dynamic-partition insert)
* INSERT INTO TABLE ... VALUES
#### 多表插入
```
FROM source
INSERT OVERWRITE TABLE stations_by_year
SELECT year, COUNT(DISTINCT station)
GROUP BY year
INSERT OVERWRITE TABLE records_by_year
SELECT year, COUNT(1)
GROUP BY year
INSERT OVERWRITE TABLE good_records_by_year
SELECT year, COUNT(1)
WHERE temperature != 9999 AND quality IN (0, 1, 4, 5, 9)
GROUP BY year
```
* 把 INSERT 语句倒过来，把 FROM 子句放在最前面，查询的效果是一样的
* 可在同一个查询中使用多个 INSERT 子句。此多表插入(multitable insert)方法比多个单独的INSERT语句效率更高
    * 只需要扫描一遍源表就可以生成多个不相交的输出
#### CREATE TABLE ... AS SELECT 语句
```
CREATE TABLE target
AS
SELECT col1, col2
FROM source;
```
* 新表的列的定义是从 SELECT 子句所检索的列导出的
    * 上面的查询中，target表有两列，分别名col1和col2，数据类型和源表中对应的列相同
* CAST操作是原子的
### 表的修改
* Hive使用读时模式(schema on read)，创建表后对表定义的修改非常灵活
    * 由用户确保修改数据以符合新的结构
```
ALTER TABLE source RENAME TO target;
```
* 重命名表
    * 更新表的元数据
    * 托管表：把表目录移到新名称所对应的目录下
        * /user/hive/warehouse/source 重命名为 /user/hive/warehouse/target
    * 外部表：只更新元数据

```
ALTER TABLE target ADD COLUMNS (col3 STRING);
```
* 添加一个新列
    * 新的列col3添加在已有(非分区)列的后面
    * 数据文件没有被更新，查询col3的所有值返回空值null(原文件无额外的字段)
    * 更常用的做法：创建一个定义了新列的新表，使用 SELECT 语句把数据填充进去
* 更进一步了解修改表的结构，包括添加或丢弃分区、修改和替换列，修改表和SerDe的属性，访问Hive的英文维基页面(http://bit.ly/data_def_lang)
### 表的丢弃
* DROP TABLE 语句用于删除表的数据和元数据
    * 外部表只删除元数据
* TRUNCATE TABLE 语句删除表内的所有数据，保留表的定义
    * 对外部表无作用，在Hive的shell环境下使用 dfs -rmr 命令直接删除外部表目录
    * CREATE TABLE new_table LIKE existing_table; 使用 LIKE 关键字创建一个与第一个表模式相同的新表

## 查询数据
* 讨论如何使用 SELECT 语句的各种形式从Hive中检索数据

### 排序和聚集
* Hive中可以使用标准的 ORDER BY 子句对数据进行排序
    * ORDER BY 将对输入执行并行全排序
* Hive的非标准的扩展 SORT BY 为每个reducer产生一个排序文件
    * 不需要结果是全局排序的情况下可用
    * DISTRIBUTE BY 某些情况下，需要某个特定的行应该到哪个reducer，其目的通常是为了执行后续的聚集操作
    ```
    hive> FORM records2
        > SELECT year, temperature
        > DISTRIBUTE BY year
        > SORT BY year ASC, temperature DESC;
    1949    111
    1949    78
    1950    22
    1950    0
    1950    -11
    ```
    * 如果 SORT BY 和 DISTRIBUTE BY 中所用的列相同，可以缩写为 CLUSTER BY 以便同时指定两者所用的列
        * CLUSTER BY 不能指定排序为asc或desc的规则，只能是desc倒序排列

### MapReduce 脚本
* TRANSFORM、 MAP 和 REDUCE 子句可以在HIve中调用外部脚本或程序
```
#!/usr/bin/env python

import re
import sys

for line in sys.stdin:
  (year, temp, q) = line.strip().split()
  if (temp != "9999" and re.match("[01459]", q)):
    print "%s\t%s" % (year, temp)
```
* 上述是过滤低质量气象记录的python脚本，Hive中可以采用下述语句使用这个脚本
```
hive> ADD FILE /Users/tom/book-workspace/hadoop-book/ch17-hive/src/main/python/is_good_quality.py;
hive> FROM records2
    > SELECT TRANSFORM(year, temperature, quality)
    > USING 'is_good_quality.py'
    > AS year, temperature;
1950 0
1950 22
1950 -11
1949 111
1949 78
```
* 运行查询前，需要在Hive中注册脚本
    * 通过这一操作，Hive知道需要把脚本文件传输到Hadoop集群上
* 查询本身把year，temperature和quality这些字段以制表符分隔的行的形式流式传递给脚本 is_good_quality，并把制表符分隔的输出解析为year和temperature字段，最终形成查询的输出
* 如果要用查询的嵌套形式，可以指定map和reduce函数。下面用 MAP 和 REDUCE 关键字。在这两个地方用 SELECT TRANSFORM 也能达到同样的效果
```
#!/usr/bin/env python

import sys

(last_key, max_val) = (None, 0)
for line in sys.stdin:
  (key, val) = line.strip().split("\t")
  if last_key and last_key != key:
    print "%s\t%s" % (last_key, max_val)
    (last_key, max_val) = (key, int(val))
  else:
    (last_key, max_val) = (key, max(max_val, int(val)))

if last_key:
  print "%s\t%s" % (last_key, max_val)
```
```
FROM (
    FROM records2
    MAP year, temperature, quality
    USING 'is_good_quality.py'
    AS year, temperature) map_output
REDUCE year, temperature
USING 'max_temperature_reduce.py'
AS year, temperature;
```

### 连接
* 和直接使用MapReduce相比，使用Hive的一个好处在于Hive简化了常用操作
#### 内连接
```
hive> SELECT * FROM sales;
joe 2
hank 4
Ali 0
Eve 3
Hank 2
hive> SELECT * FROM things;
2 Tie
4 Coat
3 Hat
1 Scarf
hive> SELECT sales.*, things.* FROM sales JOIN things ON (sales.id = things.id);
Joe 2 2 Tie
Hank 4 4 Coat
Eve 3 3 Hat
Hank 2 2 Tie
```
* 最简单的一种连接。输入表之间的每次匹配都会在输出表里生成一行
* Hive只支持等值连接(equijion)
* 连接谓词中使用 AND 关键字分隔
* 查询中可使用多个 JOIN...ON... 子句来连接多个表
    * Hive会智能地以最少MapReduce作业数来执行连接
* Hive 允许在 SELECT 语句的 FROM 子句中列出要连接的表，而在 WHERE 子句中指定连接条件。上例等价于
    ```
    hive> SELECT sales.*, things.* FROM sales, things WHERE sales.id = things.id;
    ```
* 单个的连接用一个MapReduce作业实现
    * 多个连接条件中使用了相同的列，那么平均每个连接可以少用一个MapReduce作业来实现
* JOIN 子句中表的顺序很重要：一般最好将最大的表放在最后
    * 如何为Hive的查询规划器给出提示，可以访问Hive的维基页面
* 可以在查询前使用 EXPLAIN 关键字来查看 Hive 将为某个查询使用多少个MapReduce作业
    * 查看更详细的信息，可以在查询前使用 EXPLAIN EXTENDED
    ```
    EXPLAIN
    SELECT sales.*, things.*
    FROM sales JOIN things ON (sales.id = things.id);
    ```
* Hive 目前使用基于规则的查询优化器来确定查询是如何执行的
    * 0.14.0开始，也可以使用基于代价的优化器

#### 外连接
```
hive> SELECT sales.*, things.* FROM sales LEFT OUTER JOIN things ON (sales.id = things.id);
Joe 2 2 Tie
Hank 4 4 Coat
Ali 0 NULL NULL
Eve 3 3 Hat
Hank 2 2 Tie
hive> SELECT sales.*, things.* FROM sales RIGHT OUTER JOIN things ON (sales.id = things.id);
Joe 2 2 Tie
Hank 2 2 Tie
Hank 4 4 Coat
Eve 3 3 Hat
NULL NULL 1 Scarf
hive> SELECT sales.*, things.* FROM sales FULL OUTER JOIN things ON (sales.id = things.id);
Ali 0 NULL NULL
NULL NULL 1 Scarf
Joe 2 2 Tie
Hank 2 2 Tie
Eve 3 3 Hat
Hank 4 4 Coat
```
* 外连接可以找到连接表中不能匹配的数据行。上述为左外连接、右外连接和全外连接(full outer join)的示例
#### 半连接
```
SELECT *
FROM things
WHERE things.id IN (SELECT id from sales);
```
等价于
```
hive> SELECT * FROM things LEFT SEMI JOIN sales ON (sales.id = things.id);
2 Tie
4 Coat
3 Hat
1 Scarf
```
* LEFT SEMI JOIN 查询时必须遵循一个限制：右表只能在 ON 子句中出现

#### map 连接
```
SELECT sales.*, things.* 
FROM sales JOIN things ON (sales.id = things.id);
```
* 如果有一个连接表小到足以放入内存，Hive就把较小的表(things)放入每个mapper的内存来执行连接操作，这就称为map连接
* 执行这个查询不适用reducer，因此此查询对 RIGHT 或 FULL OUTER JOIN 无效，因为只有在对所有输入上进行聚集的步骤才能检测到哪个数据行无法匹配
* map连接可以利用分桶的表
    * 需要用下面的语法启用优化选项：
        ```
        SET hive.optimize.bucketmapjoin=true;
        ```
### 子查询
* Hive对子查询的支持很有限，只允许子查询出现在 SELECT 语句的 FROM 子句中，或某些特殊情况下的 WHERE 子句中
```
SELECT station, year, AVG(max_temperature)
FROM (
SELECT station, year, MAX(temperature) AS max_temperature
FROM records2
WHERE temperature != 9999 AND quality IN (0, 1, 4, 5, 9)
GROUP BY station, year
) mt
GROUP BY station, year
```
### 视图
* 视图是一种用 SELECT 语句定义的虚表(virtual table)
    * 视图可以用来以一种不同于磁盘实际存储的形式把数据呈现给用户
        * 现有表的数据常常需要以一种特殊的方式进行简化和聚集以便于后期处理
    * 视图可以用来限制用户，使其只能访问被授权可以看到的子集
* Hive中，创建视图时并不执行查询和把视图物化存储到磁盘上，查询只是存储在metastore中。视图的 SELECT 语句只是在执行引用视图的语句时才执行
    * 视图要对基表进行大规模的变换，或视图的查询会频繁执行，可以选择新建一个表，手工物化它(CREATE TABLE...AS SELECT)
* Hive的视图是只读的，无法通过视图为基表加载或插入数据
* SELECT TABLES 命令的输出结果包括视图
* DESCRIBE EXTENDED view_name 命令查看某个视图的详细信息
```
CREATE VIEW valid_records
AS
SELECT *
FROM records2
WHERE temperature != 9999 AND quality IN (0, 1, 4, 5, 9);


CREATE VIEW max_temperatures (stations, year, max_temperature)
AS
SELECT stations, year, MAX(temperature) FROM valid_records
GROUP BY stations, year;

SELECT stations, year, AVG(max_temperature)
FROM max_temperatures
GROUP BY stations, year;
```
* 此查询和前面使用子查询的查询是一样的。Hive为他们所创建的MapReduce作业的个数也是一样的：都是两个，每个 GROUP BY 使用一个
    * 即使在执行时，Hive也不会在不必要的情况下物化视图
## 用户定义函数
* 编写用户定义函数(user-defined function, UDF)，Hive可以方便地插入用户写的处理代码并在查询中调用它们
* UDF必须用Java语言编写
    * 其他编程语言可以考虑使用 SELECT TRANSFORM 查询，可以让数据流经用户定义的脚本
* Hive中有三种UDF，它们所接受的输入和产生的输出的数据行的数量不同：
    * (普通)UDF
        * UDF操作作用于单个数据行，且产生一个数据行作为输出
        * 大多数函数(例如数学函数和字符串函数)都属于这一类
    * 用户定义聚集函数(user-defined aggregate function, UDAF)
        * UDAF接受多个输入数据行，并产生一个输出数据行
        * 像 COUNT 和 MAX 这样的函数
    * 用户定义表生成函数(user-defined table-generating function, UDTF)
        * UDTF作用于单个数据行，且产生多个数据行(即一个表)作为输出
### UDTF
```
CREATE TABLE arrays (x ARRAY<STRING>) ROW
FORMAT DELIMITED
  FIELDS TERMINATED BY '\001'
  COLLECTION ITEMS TERMINATED BY '\002';
``` 
* 考虑上述样式的表，只有一列x，包含的是字符串数组
```
hive> SELECT * FROM arrays;
["a", "b"]
["c", "d", "e"]
hive> SELECT explode(x) AS y FROM arrays;
a
b
c
d
e
```
* 此处使用 explode UDTF 对表进行变换。此函数为数组中的每一项输出一行
* 带 UDTF 的 SELECT 语句在使用时有一些限制(例如，不能检索额外的列表达式)，使实际使用时这些语句的用处不大
    * select id,explode(arry1) from table; 错误
    * 为此，Hive支持 LATERAL VIEW 查询(http://bit.ly/lateral_view)
        * 在使用lateral view的时候需要指定视图别名和生成的新列别名
            ```
            select id,num from table lateral view explode(array1) subview as num;
            select id,num1,num2 from table
            lateral view explode(array1) subview1 as num1
            lateral view explode(array2) subview2 as num2
            where ...;
            ```
### 写 UDF
```
package com.hadoopbook.hive;

import org.apache.commons.lang.StringUtils;
import org.apache.hadoop.hive.ql.exec.UDF;
import org.apache.hadoop.io.Text;

public class Strip extends UDF {
  private Text result = new Text();
  
  public Text evaluate(Text str) {
    if (str == null) {
      return null;
    }
    result.set(StringUtils.strip(str.toString()));
    return result;
  }
  
  public Text evaluate(Text str, String stripChars) {
    if (str == null) {
      return null;
    }
    result.set(StringUtils.strip(str.toString(), stripChars));
    return result;
  }
}
```
* 一个简单的剪除字符串尾字符的UDF
    * Hive已经有一个内置的名为trim的函数
* 上述Strip类有两个evaluate()方法：
    * evaluate(Text str)去除前导和结束的空白字符
    * evaluate(Text str, String stripChars)去除字符串尾出现的指定字符集合中的任何字符
* Hive支持在UDF中使用Java的基本数据类型(以及一些像java.util.List和java.util.Map这样的类型)
    * public String evaluate(String str)效果是一样的
    * 但是通过使用Text，可以利用对象重用的优势，增效节支
* 一个UDF必须满足两个条件：
    * 一个UDF必须是org.apache.hadoop.hive.ql.exec.UDF的子类
    * 一个UDF必须至少实现了evaluate()方法
        * evaluate()方法不是由接口定义的，因为它可接受的参数的个数、它们的数据类型及其返回值的数据类型都是不确定的
        * Hive会检查UDF，看能否找到和函数调用相匹配的evaluate()方法
* 为了在Hive中使用UDF，需要把编译后的Java类打包成一个JAR文件，接下来，在metastore中注册这个函数并使用CREATE FUNCTION语句为它起名：
    ```
    CREATE FUNCTION strip AS 'com.hadoopbook.hive.Strip'
    USING JAR '/path/to/hive-examples.jar';
    ```
    * 本地使用Hive只需要一个本地文件路径就足够了，但在集群上，应当将JAR文件复制到HDFS中，并在 USING JAR 子句中使用HDFS的URI
    * 如此，可以想使用内置函数一样使用UDF：
        ```
        hive> SELECT strip(' bee ') FROM dummy;
        bee
        hive> SELECT STRIP('banana', 'ab') FROM dummy;
        nan
        ```
* 删除此函数，可以使用 DROP FUNCTION 语句
    ```
    DROP FUNCTION strip;
    ```
* TEMPORARY 关键字可以创建一个仅在Hive会话期间有效的函数，即此函数并没有在metastore中持久化存储
    ```
    ADD JAR /path/to/hive-examples.jar;
    CREATE TEMPORARY FUNCTION strip AS 'com.hadoopbook.hive.Strip'
    ```
    * 使用临时函数时，最好在主目录中创建一个.hiverc文件，以包含定义这些UDF的命令，而这个文件会在每个Hive会话开始时自动运行
* 可以在启动时指定查找附加JAR文件的路径，此路径会被加入Hive的类路径(也包括任务的类路径)
    * 有两种指明路径的办法：
        * 在hive命令后传递 --auxpath 选项：
            * % hive --auxpath /path/to/hive-examples.jar
        * 运行Hive前设置HIVE_AUX_JARS_PATH环境变量
    * 附加路径可以是一个用逗号分隔的JAR文件路径列表或包含JAR文件的目录

### 写 UDAF
```
package com.hadoopbook.hive;

import org.apache.hadoop.hive.ql.exec.UDAF;
import org.apache.hadoop.hive.ql.exec.UDAFEvaluator;
import org.apache.hadoop.io.IntWritable;

public class Maximum extends UDAF {

  public static class MaximumIntUDAFEvaluator implements UDAFEvaluator {
    
    private IntWritable result;
    
    public void init() {
      System.err.printf("%s %s\n", hashCode(), "init");
      result = null;
    }

    public boolean iterate(IntWritable value) {
      System.err.printf("%s %s %s\n", hashCode(), "iterate", value);
      if (value == null) {
        return true;
      }
      if (result == null) {
        result = new IntWritable(value.get());
      } else {
        result.set(Math.max(result.get(), value.get()));
      }
      return true;
    }

    public IntWritable terminatePartial() {
      System.err.printf("%s %s\n", hashCode(), "terminatePartial");
      return result;
    }

    public boolean merge(IntWritable other) {
      System.err.printf("%s %s %s\n", hashCode(), "merge", other);
      return iterate(other);
    }

    public IntWritable terminate() {
      System.err.printf("%s %s\n", hashCode(), "terminate");
      return result;
    }
  }
}
```

```
hive> CREATE TEMPORARY FUNCTION maximum AS 'com.hadoopbook.hive.Maximum';
hive> SELECT maximum(temperature) FROM records;
111
```
* 聚集函数比普通的UDF难写。值是在块内进行聚集的(这些块可能分布在很多任务重)，从而实现时要能够把部分的聚集值组合成最终结果
* UDAF 必须是org.apache.hadoop.hive.ql.exec.UDAF的子类，且包含一个或多个嵌套的，实现org.apache.hadoop.hive.ql.exec.UDAFEvaluator的静态类
    * 示例中只有MaximumIntUDAFEvaluator一个嵌套类，可以添加更多的计算函数(MaximumLongUDAFEvaluator、MaximumFloatUDAFEvaluator)来提供长整型、浮点型等类型最大值的UDAF的重载
    * 一个计算函数必须实现下面5个方法：
        * init()
            * 负责初始化计算函数并重设它的内部状态
        * iterate()
            * 每次对一个新值进行聚集计算时都会调用iterate()方法。计算函数要根据聚集计算的结果更新其内部状态
            * iterate()接受的参数和Hive中被调用的参数是对应的
        * terminatePartial()
            * Hive需要部分聚集结果时会调用terminatePartial()方法
            * 此方法必须返回一个封装了聚集计算当前状态的对象
        * merge()
            * 在Hive决定要合并一部分聚集值和另一部分聚集值时会调用merge()方法
            * 接受一个对象作为输入。这个对象的类型必须和terminatePartial()方法的返回类型一致
        * terminate()
            * Hive需要最终聚集结果时会调用terminate()方法
            * 计算函数需要把状态作为一个值返回

* 下图显示了计算函数的处理流程
    ![avatar](https://github.com/zXingchu/bigdata/blob/master/hive/包含UDAF部分结果的数据流.png "包含UDAF部分结果的数据流")

#### 更复杂的UDAF
* 前面的示例中部分聚集结果可以使用和最终结果相同的类型(IntWritable)来表示。更复杂的聚集函数，情况并非如此，如计算均值
```
package com.hadoopbook.hive;

import org.apache.hadoop.hive.ql.exec.UDAF;
import org.apache.hadoop.hive.ql.exec.UDAFEvaluator;
import org.apache.hadoop.hive.serde2.io.DoubleWritable;

public class Mean extends UDAF {

  public static class MeanDoubleUDAFEvaluator implements UDAFEvaluator {
    public static class PartialResult {
      double sum;
      long count;
    }
    
    private PartialResult partial;

    public void init() {
      partial = null;
    }

    public boolean iterate(DoubleWritable value) {
      if (value == null) {
        return true;
      }
      if (partial == null) {
        partial = new PartialResult();
      }
      partial.sum += value.get();
      partial.count++;
      return true;
    }

    public PartialResult terminatePartial() {
      return partial;
    }

    public boolean merge(PartialResult other) {
      if (other == null) {
        return true;
      }
      if (partial == null) {
        partial = new PartialResult();
      }
      partial.sum += other.sum;
      partial.count += other.count;
      return true;
    }

    public DoubleWritable terminate() {
      if (partial == null) {
        return null;
      }
      return new DoubleWritable(partial.sum / partial.count);
    }
  }
}
```
* 上面的示例中，部分聚集结果用一个嵌套的静态类struct实现，由于使用了Hive能够处理的字段类型(Java原子数据类型)，Hive能够自己对这个类进行序列化和反序列化

## 延伸
* Edward Capriolo&Dean Wampler&Jason Rutherglen\<\<Programming Hive\>\> http://shop.oreilly.com/product/0636920023555.do