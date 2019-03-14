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

### 托管表和外部表












