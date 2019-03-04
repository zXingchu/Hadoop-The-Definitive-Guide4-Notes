# HBase 简单介绍

## HBase 基础
* HBase是一个在HDFS上开发的面向列的分布式数据库，适用于实时的随机访问超大规模数据集
* HBase是一个开源项目，是横向扩展的
* HBase是一个数据模型，类似于谷歌的大表设计，可以提供快速随机访问海量结构化数据，利用了HDFS提供的容错能力
* HBase在Hadoop的文件系统之上，并提供了读写访问,使用HBase可以在HDFS读取消费/随机访问数据
* 绝大多数数据库存储和检索的实现特别是关系型数据库技术的变种，不是为大国莫可伸缩的分布式处理设计的
    * 复制(replication)和分区(partitioning)让数据库能从单个节点扩展出去
    * 这些附加技术大都属于“事后”解决办法。难以安装和维护，往往牺牲一些重要的RDBMS特性
    * 在一个“扩展的”RDBMS上。连接、复杂查询、触发器、视图以及外键约束这些功能运行开销大甚至无法用
* HBase自底向上地进行构建，能够简单地通过增加节点来达到线性扩展。从另一个方向解决了可伸缩的问题
    * HBase不是关系型数据库，不支持SQL
    * 特定问题空间可以做到RDBMS不能做的事：在廉价硬件构成的集群上管理超大规模的稀疏表
    * 一个典型应用：webtable
* HBase的特点
    * HBase线性可扩展
    * 具有自动故障支持
    * 提供了一致的读取和写入
    * 集成了Hadoop，作为源和目的地
    * 客户端方便的Java API
    * 提供了跨集群数据复制。
### HBase 和 HDFS
| HDFS | HBase |
| ------ | ------ |
| HDFS是适于存储大容量文件的分布式文件系统 | HBase是建立在HDFS之上的数据库 |
| HDFS不支持快速单独记录查找 | HBase提供在较大的表快速查找 |
| 它提供了高延迟批量处理;没有批处理概念 | 它提供了数十亿条记录低延迟访问单个行记录(随机存取)|
| 它提供的数据只能顺序访问 | HBase内部使用哈希表和提供随机接入，并且其存储索引，可将在HDFS文件中的数据进行快速查找 |
### 面向列和面向行
面向列的数据库是存储数据表作为数据列的部分，而不是作为行数据
| 行式数据库	| 列式数据库 |
| ------ | ------ |
| 它适用于联机事务处理(OLTP) | 它适用于在线分析处理(OLAP) |
| 这样的数据库被设计为小数目的行和列 | 面向列的数据库设计的巨大表 |
### HBase 和 RDBMS
| HBase	| RDBMS |
| ------ | ------ |
| HBase无模式，它不具有固定列模式的概念;仅定义列族 | RDBMS有它的模式，描述表的整体结构的约束 |
| 它专门创建为宽表。 HBase是横向扩展。	这些都是细而专为小表 | 很难形成规模 |
| 没有任何事务存在于HBase |	RDBMS是事务性的 |
| 它反规范化的数据 | 它具有规范化的数据 |
| 它用于半结构以及结构化数据是非常好的 | 用于结构化数据非常好 |
### 背景
* Hadoop的限制
    * Hadoop只能执行批量处理，并且只以顺序方式访问数据
    * 这意味着必须搜索整个数据集，即使是最简单的搜索工作
    * 在这一点上，需要一个新的解决方案，访问数据中的任何点(随机访问)单元。
* Hadoop随机存取数据库
    * 应用程序，如HBase, Cassandra, couchDB, Dynamo和MongoDB都是一些存储大量数据和以随机方式访问数据的数据库
* 由Powerset公司的Chad Walters和Jim Kelleman在2006年末发起
* 起源于Google刚刚发布的BigTable
* 第一个发布版本是2007-10和Hadoop 0.15.0捆绑在一起发布的
* 2010-5 HBase从Hadoop子项目升级成Apache顶层项目

## 概念
### 数据模型
* 应用把数据存放在带标签的表中
* 表由行和列组成
    * 表中的行的键是字节数组的
* 表格的“单元格”(cell)由行和列的坐标交叉决定
    * 有版本，默认版本号自动分配，为HBase插入单元格时的时间戳
    * 单元格的内容是未解释的字节数组
* 表中的行的键也是字节数组。表中的行根据行的键值(主键)进行字节序排序。所有对表的访问都要通过表的主键
    * 理论上任何东西都可通过表示成字符串或将二进制形式转化为长整型或直接对数据结构进行序列化，来作为键值
* 行中的列被分成“列族”(column family)
    * 同一个列族的所有成员具有相同的前缀。如info\:format和info\:geo
    * 前缀必须由“可打印的”(printable)字符组成。修饰性的结尾字符(列族修饰符)，可以为任意字节
    * 列族和修饰符之间始终以冒号(:)分隔
    * 表的列族必须作为表模式定义的一部分预先给出，新的列族成员可以随后按需加入，如添加新的列info\:camera
    * 物理上，所有的列族成员都一起存放在文件系统中，HBase更准确的说是一个面向列族的存储器
    * 由于调优和存储都是在列族层次上进行的，最好使所有列族成员都有相同的访问模式(access pattern)
* 表是行的集合，行是列族的集合，列族是列的集合，列是键值对的集合

![avatar](https://github.com/zXingchu/bigdata/blob/master/hbase/用于描述存储照片的表的HBase数据模型.png "用于描述存储照片的表的HBase数据模型")
#### 区域
* HBase自动把表水平划分成区域(region)
    * 每个区域由表中行的子集构成
    * 每个区域由它所属于的表、它所包含的第一行及其最后一行(不包括这行)来表示
    * 区域是在HBase集群上分布数据的最小单位
* 一开始一个表只有一个区域。随着区域开始变大，超出设定的大小阈值，便会在某行的边界上把表分成两个大小基本相同的新分区
    * 第一次划分之前，所有加载的数据都放在原始区域所在的那台服务器上
    * 用这种方式，大表被放在服务器集群上，每个节点都负责管理表所有区域的一个子集
    * 表的加载也是用这种方法把数据分布到各个节点
    * 在线的所有区域按次序排列就构成了表的所有内容

#### 加锁
* 无论对行进行访问的事务牵涉多少列，对行的更新都是“原子的”(atomic)
    * 这使得“加锁模型”(locking model)能够保持简单

### 实现
* HDFS和YARN是由客户端、从属机(slave)和协调主控机(master)组成，HBase采用相同的模型
    * HDFS的namenode和datanode，YARN的资源管理器和节点管理器
* HBase用一个master节点协调管理一个或多个regionserver从属机
    * HBase主控机(master)负责启动(bootstrap)一个全新的安装，把区域分配给注册的regionserver，恢复regionserver的故障。负载很轻
    * regionserver负责零个或多个区域的管理以及响应客户端的读/写请求。还负责区域的划分并通知HBase master有了新的子区域(daughter region)，如此，主控机就可以把父区域设为离线，并用子区域替换父区域

![avatar](https://github.com/zXingchu/bigdata/blob/master/hbase/HBase集群的成员.png "HBase集群的成员")

* HBase依赖于ZooKeeper
    * 默认情况下，它管理一个ZooKeeper实例，作为集群的“权威机构”(authority)
    * 也可以通过配置来使用已有的ZooKeeper集群
    * ZooKeeper集合体(ensemble)负责管理诸如hbase:meta目录表的位置以及当前集群主控机地址等重要信息
    * 如果在区域的分配过程中有服务器崩溃，就可以通过ZooKeeper来进行分配的协调。在ZooKeeper上管理分配事务的状态有助于在恢复时能够从崩溃服务器遗留的状态开始继续分配
    * 在启动一个客户端到HBase集群的连接时，客户端必须至少拿到到集群所传递的ZooKeeper集合体的位置
        * 这样，客户端才能访问ZooKeeper的层次结构，从而了解集群的属性，例如服务器的位置

* regionserver从属机节点列在HBase的conf/regionserver文件中
    * 类似于在Hadoop中可以通过etc/hadoop/slaves文件查看datanode和节点管理器
* 启动和结束服务的脚本和Hadoop类似，使用相同的基于SSH的机制来运行远程命令
* 集群的站点配置(site-specific configuration)在HBase的conf/hbase-site.xml和conf/hbase-env.sh文件中
    * 格式和Hadoop父项目中对应的格式相同
    * 对于HBase和Hadoop上相同的服务或类型，HBase实际上直接使用或继承Hadoop的实现。在无法使用或继承时，HBase会尽量遵循Hadoop的模型。HBase只是在增加特殊功能时才不遵循这一规则

* HBase 通过Hadoop文件系统API来持久化存储数据
    * 默认情况下，除非另行指明，HBase会将存储写入本地文件系统
    * 如果要使用HBase集群，首要任务通常是把HBase的存储配置为指向要使用的HDFS集群
 
#### 运行中的HBase
* HBase内部保留名为hbase:meta的特殊目录表(catalog table)
    * 它们维护着当前集群上所有区域的列表、状态和位置
    * hbase:meta表中的项使用区域名作为键
        * 区域名由所属的表名、区域的起始行、区域的创建时间以及其整体进行的MD5哈希值组成
            * MD5哈希值即对表名、起始行、创建的时间戳进行哈希后的结果
            * 如，表TestTable中起始行为xyz的区域的名称如下：
                * TestTablee,xyz.1279729913622.1b6e6176fb8d8aa88fd4ab6bc80247ece.
                * 在表名、起始行、时间戳中间用逗号分隔额。MD5哈希值则使用前后两个句号包围
* 如前所述，行的键是排序的。因此，要查找一个特定行所在的区域只要在目录表中找到第一个键大于或等于给定行键的项即可
    * 区域变化时，即分裂、禁用/启用(disable/enable)、删除、为负载均衡重新部署区域或由于regionserver崩溃而重新部署区域时，目录表会进行相应的更新。这样，集群上所有区域的状态信息就能保持是最新的
* 新连接到ZooKeeper集群上的客户端
    * 首先查找hbase:meta的位置
    * 然后客户端通过查找合适的hbase:meta区域来获取用户空间区域所在节点及其位置
    * 接着，客户端就可以直接和管理那个区域的regionserver进行交互
* 每个行操作可能要访问三次远程节点，为了节省这些代价，客户端会缓存它们遍历hbase:meta时获取的信息
    * 需要缓存的有位置信息，用户空间区域的开始行和结束行
        * 如此，以后不需要访问hbase:meta表也能得知区域存放的位置
    * 客户端在碰到错误之前会一直使用所缓存的项
        * 当发生错误
            * 区域被移动了，客户端会再去查看hbase:meta获取区域的新位置
            * hbase:meta区域也被移动了，客户端会重新查找
* 到达Regionserver的写操作
    * 首先追加到“提交日志”(commit log)
    * 然后加入内存中的memstore
        * 如果memstore满，它的内容会被“刷入”(flush)文件系统
    * 提交日志存放在HDFS中
        * 因此即使一个regionserver崩溃，提交日志仍然可用
    * 如果发现一个regionserver不能访问(通常因为服务器的znode在ZooKeeper中过期)，主控机会根据区域对死掉的regionserver的提交日志进行分割
        * 重新分配后，在打开并使用死掉的regionserver上的区域之前，这些区域会找到属于它们的从被分割提交日志中得到的文件，其中包含还没有被持久化存储的更新。这些更新会被“重做”(replay)以使区域恢复到服务器失败前的状态
* 在读的时候
    * 首先查看区域的memstore
    * 如果在memstore中找到了需要的版本，查询就结束了
    * 否则，需要按次序从新到旧检查“刷新文件”(flush file)，直到查找到满足查询的版本，或所有刷新文件都处理完为止
* 有一个后台进程负责在刷新文件个数到达一个阈值时压缩它们
    * 它把多个文件重新写入一个文件
        * 因为读操作检查的文件越少，它的执行效率越高
    * 在压缩(compaction)时，进程会清理掉超出模式所设最大值的版本以及删除单元格或标识单元格为过期
* 在regionserver上，另外有一个独立的监控着刷新文件的大小，一旦文件大小超出预先设定的最大值，便对区域进行分割

## 安装
* Apache Download Mirror(http://www.apache.org/dyn/closer.cgi/hbase/)，下载，本地系统解压
    ```
    % tar xzf hbase-x.y.z.tar.gz
    ```
* Java
    * 设置JAVA_HOME环境变量
    * 编辑HBase的conf/hbase-env.sh，并指明JAVA_HOME变量的值，从而设置所使用的Java安装
* 把HBase的二进制文件目录加入命令行路径中
    ```
    % export HBASE_HOME=~/sw/hbase-x.y.z
    % export PATH=$PATH:$HBASE_HOME/bin
    ```
* 获取HBase的选项列表
    ```
    % hbase
    ```
* 默认日志目录${HBASEHOME}/logs
#### 测试驱动
* 启动一个使用本地文件系统/tmp目录作为持久化存储的HBase临时实例
    ```
    % start-hbase.sh
    ```
* 默认情况下，HBase会被写入到/${java.io.tmpdir}/hbase-${user.name}中
    * ${java.io.tmpdir}通常被映射为/tmp，可通过设置hbase-site.xml中的hbase.tmp.dir对HBase进行配置
* 在独立模式下，HBase主控机、regionserver和ZooKeeper实例都是在同一个JVM中运行的
* 要管理HBase实例，用以下命令启动HBase的shell环境
    * 这将启动一个加入了一些HBase特有命令的JRuby IRB解释器
    * help 查看已分组的shell环境的命令列表
    * help COMMAND_GROUP 查看某一类命令的帮助
    * help COMMAND 获得某个特定命令的帮助信息和用法示例
    * 命令使用Ruby的格式来指定列表和目录
    ```
    % hbase shell
    ```
##### 创建一个简单的表，添加一些数据，再把表清空
* 新建一个表
    * 首先必须为表命名，并为其定义模式
        * 模式包含表的属性和列族的列表
        * 列族本身也有属性，可以在定义模式时依次定义
        * 模式可以被修改，修改时把表设为“离线”(offline)即可
        * shell环境中diable命令把表设为离线，alter命令进行必要修改，enable命令重新把表设为“在线”(online)
    * 新建一个名为test的表，只包含一个名为data的列，表和列族属性均为默认值
        ```
        hbase> create 'test', 'data'
        ```
* 验证新表是否创建成功，使用list命令，输出用户空间中的所有表
    ```
    hbase> list
    ```
* 在列族data中三个不同的行和列上插入数据，获取第一行，后列出表的内容
    * 注意如何在添加三个新列的时候不修改模式的
    ```
    hbase> put 'test', 'row1', 'data:1', 'value1'
    hbase> put 'test', 'row2', 'data:2', 'value2'
    hbase> put 'test', 'row3', 'data:3', 'value3'
    hbase> get 'test', 'row1'
    
    hbase> scan 'test'
    ```
* 移除这个表，首先把它设为禁用，然后删除
    ```
    hbase> disable 'test'
    hbase> drop 'test'
    hbase> list
    ```
* 运行以下命令关闭HBase实例
    ```
    % stop-hbase.sh
    ```
* 设置分布式的HBase，并把它指向正运行的HDFS
    * http://hbase.apache.org/book/configuration.html

## 客户端
* 和HBase集群进行交互，有很多种不同的客户端可供选择
### Java
* HBase和Hadoop一样，用Java开发的
* 上述测试驱动在shell环境下运行的java实现版本
    * 基本的表管理与访问
    ```
    import java.io.IOException;
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.hbase.HBaseConfiguration;
    import org.apache.hadoop.hbase.HColumnDescriptor;
    import org.apache.hadoop.hbase.HTableDescriptor;
    import org.apache.hadoop.hbase.TableName;
    import org.apache.hadoop.hbase.client.Get;
    import org.apache.hadoop.hbase.client.HBaseAdmin;
    import org.apache.hadoop.hbase.client.HTable;
    import org.apache.hadoop.hbase.client.Put;
    import org.apache.hadoop.hbase.client.Result;
    import org.apache.hadoop.hbase.client.ResultScanner;
    import org.apache.hadoop.hbase.client.Scan;
    import org.apache.hadoop.hbase.util.Bytes;
    
    public class ExampleClient {
    
      public static void main(String[] args) throws IOException {
        Configuration config = HBaseConfiguration.create();
        // Create table
        HBaseAdmin admin = new HBaseAdmin(config);
        try {
          TableName tableName = TableName.valueOf("test");
          HTableDescriptor htd = new HTableDescriptor(tableName);
          HColumnDescriptor hcd = new HColumnDescriptor("data");
          htd.addFamily(hcd);
          admin.createTable(htd);
          HTableDescriptor[] tables = admin.listTables();
          if (tables.length != 1 &&
              Bytes.equals(tableName.getName(), tables[0].getTableName().getName())) {
            throw new IOException("Failed create of table");
          }
          // Run some operations -- three puts, a get, and a scan -- against the table.
          HTable table = new HTable(config, tableName);
          try {
            for (int i = 1; i <= 3; i++) {
              byte[] row = Bytes.toBytes("row" + i);
              Put put = new Put(row);
              byte[] columnFamily = Bytes.toBytes("data");
              byte[] qualifier = Bytes.toBytes(String.valueOf(i));
              byte[] value = Bytes.toBytes("value" + i);
              put.add(columnFamily, qualifier, value);
              table.put(put);
            }
            Get get = new Get(Bytes.toBytes("row1"));
            Result result = table.get(get);
            System.out.println("Get: " + result);
            Scan scan = new Scan();
            ResultScanner scanner = table.getScanner(scan);
            try {
              for (Result scannerResult : scanner) {
                System.out.println("Scan: " + scannerResult);
              }
            } finally {
              scanner.close();
            }
            // Disable then drop the table
            admin.disableTable(tableName);
            admin.deleteTable(tableName); 
          } finally {
            table.close();
          }
        } finally {
          admin.close();
        }
      }
    }
    ```
    * 在这个类中，首先让HBaseConfiguration类来创建Configuration对象
        * 这个类会返回一个Configuration，其中已经读入了程序classpath下hbase-site.xml和hbase-default.xml文件中的HBase配置信息
        * 这个Configuration接下来会被用于创建HBaseAdmin和HTable实例，Configuration实例将这些类指向了执行这些代码的集群
            * HBaseAdmin用于管理HBase集群，添加和丢弃表
            * HTable则用于访问指定的表
    * 要创建一个表，需要首先创建一个HBASEAdmin的实例，来创建表
        * 示例中，表的模式是默认的，可以用HTableDescriptor和HColumnDescriptor的方法修改表的模式
    * 要对一个表进行操作，需要新建一个HTable的实例，并把Configuration和HColumnDescriptor和表名传递给它
    * 然后，在循环中创建Put对象，以便将数据插入到表
        * Put把单个的单元格值valuen放入名为rown的行的名为data\:n的列上
        * 上面的代码使用了HBase的Bytes实用类来把标识符和值转换为HBase所需的字节数组
    * 接着，新建一个Get对象来获取和打印刚添加的第一行
    * 然后，再使用Scan对象来扫描新建的表，并打印扫描结果
    * 最后，首先禁用表，接着删除它
    * 新的客户端API下，HBASEAdmin和HTable的构造函数已被弃用。客户端应该使用新的ConnectionFactory类创建一个Connection对象，根据需要调用getAdmin()或getTable()来检索Admin或Table实例
    ```
    import java.io.IOException;
    import org.apache.hadoop.conf.Configuration;
    import org.apache.hadoop.hbase.HBaseConfiguration;
    import org.apache.hadoop.hbase.HColumnDescriptor;
    import org.apache.hadoop.hbase.HTableDescriptor;
    import org.apache.hadoop.hbase.TableName;
    import org.apache.hadoop.hbase.client.Admin;
    import org.apache.hadoop.hbase.client.Connection;
    import org.apache.hadoop.hbase.client.ConnectionFactory;
    import org.apache.hadoop.hbase.client.Get;
    import org.apache.hadoop.hbase.client.Put;
    import org.apache.hadoop.hbase.client.Result;
    import org.apache.hadoop.hbase.client.ResultScanner;
    import org.apache.hadoop.hbase.client.Scan;
    import org.apache.hadoop.hbase.client.Table;
    import org.apache.hadoop.hbase.util.Bytes;
    
    /**
    * HBase 1.0 version of ExampleClient that uses {@code Connection},
    * {@code Admin} and {@code Table}.
    */
    public class NewExampleClient {
    
      public static void main(String[] args) throws IOException {
        Configuration config = HBaseConfiguration.create();
        Connection connection = ConnectionFactory.createConnection(config);
        try {
          // Create table
          Admin admin = connection.getAdmin();
          try {
            TableName tableName = TableName.valueOf("test");
            HTableDescriptor htd = new HTableDescriptor(tableName);
            HColumnDescriptor hcd = new HColumnDescriptor("data");
            htd.addFamily(hcd);
            admin.createTable(htd);
            HTableDescriptor[] tables = admin.listTables();
            if (tables.length != 1 &&
                Bytes.equals(tableName.getName(), tables[0].getTableName().getName())) {
                throw new IOException("Failed create of table");
            }
            // Run some operations -- three puts, a get, and a scan -- against the table.
            Table table = connection.getTable(tableName);
            try {
              for (int i = 1; i <= 3; i++) {
                byte[] row = Bytes.toBytes("row" + i);
                Put put = new Put(row);
                byte[] columnFamily = Bytes.toBytes("data");
                byte[] qualifier = Bytes.toBytes(String.valueOf(i));
                byte[] value = Bytes.toBytes("value" + i);
                put.add(columnFamily, qualifier, value);
                table.put(put);
              }
              Get get = new Get(Bytes.toBytes("row1"));
              Result result = table.get(get);
              System.out.println("Get: " + result);
              Scan scan = new Scan();
              ResultScanner scanner = table.getScanner(scan);
              try {
                for (Result scannerResult: scanner) {
                  System.out.println("Scan: " + scannerResult);
                }
              } finally {
                scanner.close();
              }
              // Disable then drop the table
              admin.disableTable(tableName);
              admin.deleteTable(tableName);
            } finally {
              table.close();
            }
          } finally {
            admin.close();
          }
        } finally {
          connection.close();
        }
      }
    }
    ```
* 大多数HBase类都位于org.apache.hadoop.hbase和org.apache.hadoop.hbase.client包中
* 编译程序和运行程序
    ```
    % mvn package
    % export HBASE_CLASSPATH=hbase-examples.jar
    % hbase ExampleClient
    ```
* 扫描器
    * HBase扫描器(scanner)和传统数据库中的游标(cursor)或Java中的迭代器(iterator)类似。不同之处在于扫描器使用后需要关闭
    * 用户使用已设置的Scan对象实例作为scan参数，调用getScanner()，获取HBase中一个表上的扫描器
    * 通过Scan实例，可传递扫描开始位置和结束位置的数据行、返回结果中要包含的列以及运行在服务器端的过滤器
    * ResultScanner接口是调用getScanner()时返回的接口，定义如下：
        ```
        public interface ResultScanner extends Closeable, Iterable<Result> {
          public Result next() throws IOException;
          public Result [] next(int nbRows) throws IOException;
          public void close();
        }
        ```
    * 扫描器会在幕后每次获取100行数据，把这些结果放在客户端，并只有在当前这批结果都被取光后才再去服务器端获取下一批结果。
        * 以这种方式获取和缓存的行数是由hbase.client.scanner.caching配置选项所决定的，或可通过setCaching()方法设置Scan实例自己缓存(cache)/预取(prefetch)的行数
        * 避免把缓存值设得太高，会导致客户端用于处理一批数据的时间超出扫描器的超时时间。如果在扫描器超时之前，客户端没有能再次访问服务器，那么扫描器在服务器端所用的资源会被服务器端的垃圾收集器自动回收
        * 默认的扫描器超时时间为60s，在hbase.client.scanner.timeout.period中设置
        * 扫描器超时，客户端会收到一个UnknownScannerException异常
### MapReduce
* org.apache.hadoop.hbase.mapreduce包中的类和工具有利于将HBase作为MapReduce作业的源/输出
* TableInputFormat类可以再区域的边界进行分割，使map能够拿到单个的区域进行处理
* TableOutputFormat将把reduce的结果写入HBase
* 一个计算HBase表中行数的MapReduce应用程序，使用TableInputFormat来运行一个map任务，以计算行数
     ```
     import org.apache.hadoop.conf.Configured;
     import org.apache.hadoop.hbase.HBaseConfiguration;
     import org.apache.hadoop.hbase.client.Result;
     import org.apache.hadoop.hbase.client.Scan;
     import org.apache.hadoop.hbase.filter.FirstKeyOnlyFilter;
     import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
     import org.apache.hadoop.hbase.mapreduce.TableMapReduceUtil;
     import org.apache.hadoop.hbase.mapreduce.TableMapper;
     import org.apache.hadoop.mapreduce.Job;
     import org.apache.hadoop.mapreduce.lib.output.NullOutputFormat;
     import org.apache.hadoop.util.Tool;
     import org.apache.hadoop.util.ToolRunner;
     
     public class SimpleRowCounter extends Configured implements Tool 
     
       static class RowCounterMapper extends TableMapper<ImmutableBytesWritable, Result> {
         public static enum Counters { ROWS }
         
         @Override
         public void map(ImmutableBytesWritable row, Result value, Context context) {
           context.getCounter(Counters.ROWS).increment(1);
         }
       }
       
       @Override
       public int run(String[] args) throws Exception {
         if (args.length != 1) {
           System.err.println("Usage: SimpleRowCounter <tablename>");
           return -1;
         }
         String tableName = args[0];
         Scan scan = new Scan();
         scan.setFilter(new FirstKeyOnlyFilter());
         
         Job job = new Job(getConf(), getClass().getSimpleName());
         job.setJarByClass(getClass());
         TableMapReduceUtil.initTableMapperJob(tableName, scan,
           RowCounterMapper.class, ImmutableBytesWritable.class, Result.class, job);
         job.setNumReduceTasks(0);
         job.setOutputFormatClass(NullOutputFormat.class);
         return job.waitForCompletion(true) ? 0 : 1;
       }
       
       public static void main(String[] args) throws Exception {
         int exitCode = ToolRunner.run(HBaseConfiguration.create(),
           new SimpleRowCounter(), args);
         System.exit(exitCode);
       }
     }
     ```
     * 嵌套类RowCounterMapper是HBase的TableMapper抽象类的一个子类
         * TableMapper是org.apache.hadoop.mapreduce.Mapper的“特化”(specialization)。
         * 它设定map的输入类型由TableInputFormat来传递
         * 输入的键为ImmutableBytesWritable对象(行键)，值为Result对象(扫描的行结果)
     * 在run()方法中，创建了一个扫描对象，设置了扫描过滤器
         * 扫描对象通过调用TableMapReduceUtil.initTableMapJob()实用方法来对作业进行配置，除了设置输入格式到TableInputFormat之外，也可做其他事情如设置要使用的map类
         * FirstKeyOnlyFilter告诉服务器，让它在运行服务器端任务时只用每行的第一个单元格来填充mapper中的Result对象
     
### REST 和 Thrift
* HBase提供了REST和Thrift接口
    * 在使用java以外的编程语言和HBase交互时，会用到这些接口
* 在所有情况下，Java服务器上都运行着一个HBase客户端实例
    * 负责协调REST和Thrift应用请求和HBase集群间的双向交互
* 详细情况参阅参考指南，http://hbase.apahce.org/book.html

## HBase 和 RDBMS 的比较
* HBase
    * 是一个分布式的、面向列的数据存储系统
    * 通过在HDFS上提供随机读/写来解决Hadoop不能处理的问题
    * 自底层设计开始即聚焦于各种可伸缩问题
        * 表可以很“高”(数十亿个数据行)
        * 表可以很“宽”(数百万个列)
    * 水平分区并在上千个普通商用机节点(commodity node)上自动复制
    * 表的模式是物理存储的直接反映，使系统有可能提供高效的数据结构的序列化、存储和检索
        * 应用程序开发者需要选择以正确的方式使用这种存储和检索方式
* RDBMS
    * 是一个遵循“Codd的12条规则“的数据库
    * 标准的RDBMS是模式固定、面向行的数据库且具有ACID性质的SQL查询引擎
    * 强调事务的“强一致性”(strong consistency)、参照完整性(referential integrity)、数据抽象与物理存储层相对独立，以及基于SQL语言的复杂查询支持
    * 可以非常容易的建立“二级索引”(secondary index)，执行复杂的内连接和外连接，执行计数、求和、排序、分组等操作，或对表、行和列中的数据进行分页存放
* 要在数据规模和并发读/写两方面中的任何一个上进行大规模向上扩展(scale up)，尤其进行分布式处理，RDBMS非常困难，且扩展通常要求打破Codd的规则，降低易用性
### HBase
HBase 具有以下特性：
* 没有真正的索引
    * 行是顺序存储的，每行中的列也是，所以不存在索引的膨胀的问题，而且插入性能和表的大小无关
* 自动分区
    * 在表的增长的时候，表会自动分裂成区域，并分布到可用的节点上
* 线性扩展和对于新节点的自动处理
    * 增加一个节点，把它指向现有集群并运行regionserver。区域自动重新进行平衡，负载均匀分布
* 普通商用硬件支持
    * 集群可以用1000~5000美金的单个节点搭建，而不需要使用单个得花5万美金的节点。RDBMS需要支持大量I/O，要求更昂贵的硬件
* 容错
    * 大量的节点意味着每个节点的重要性并不突出。不用担心单个节点失效
* 批处理
    * MapReduce集成功能使可以用全并行的分布式作业根据“数据的位置”(location awareness)来处理
## Praxis
### HDFS
* HBase使用HDFS的方式与MapReduce使用HDFS的方式截然不同
    * 在MapReduce中，首先打开HDFS文件，然后map任务流式处理任务的内容，最后关闭文件
    * 在HBase中，数据文件在启动时就被打开，并在处理过程中始终保持打开状态(为了节省每次访问操作打开文件所需的代价)
    * HBase更容易碰到MapReduce客户端不常碰到的问题

#### 文件描述符用完
* 在连接的集群上保持文件的打开状态，可能很快达到系统和Hadoop设定的限制
* 一个进程默认的文件描述符限制是1024
* 当使用的描述符个数超过文件系统的ulimit值，在日志中会看到“Too many open files”的错误信息。修正这个问题需要增加文件描述符的ulimit参数值
* 如何增加集群的ulimit值，参阅HBase参考指南，http://hbase.apache.org/book.html
#### datanode 上的线程用完
* Hadoop 1的datanode上同时运行的线程数不能超过256的限制值(dfs.datanode.max.xcievers)，会导致HBase异常运行
* Hadoop 2将默认值提高到4096
* 可以通过在hdfs-site.xml中配置dfs.datanode.max.transfer.threads(新名称)来更改设置
### 用户界面
* HBase在主控机上运行了一个Web服务器，能提供运行中集群的状态视图
    * 默认情况下，监听60010端口
    * 主界面显示了基本的属性(包括软件版本、集群负载、请求频率、集群表的列表)和加入的regionserver等
    * 在主界面点击选中regionserver会进入那个regionserver上运行的Web服务器
        * 列出了此服务器上所有区域的列表及其他基本的属性值，如使用的资源和请求频率
### 度量
* 类似Hadoop的度量(metric)系统，每过一段时间获取系统重要组件的信息，并输出到上下文(context)，HBase也有度量系统
* 相关信息可以参见HBase conf目录下的hadoop-metrics2-properties
### 计数器
* 使用HTable的incrementColumnValue()方法后，计数器每秒可以实现数千次更新
## 延伸
* O'Reilly \<\<HBase: The DefinitiveGuide\>\> http://hbase.apache.org/book.html
* Nick Dimiduk&Amandeep Khurana \<\<HBase in Action\>\> http://www.manning.com/dimidukkhurana/