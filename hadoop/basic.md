<!-- TOC -->

- [hadoop 分布式文件系统](#hadoop-%E5%88%86%E5%B8%83%E5%BC%8F%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F)
  - [Hadoop 概念](#hadoop-%E6%A6%82%E5%BF%B5)
  - [HDFS 组成架构](#hdfs-%E7%BB%84%E6%88%90%E6%9E%B6%E6%9E%84)
  - [HDFS 存储机制](#hdfs-%E5%AD%98%E5%82%A8%E6%9C%BA%E5%88%B6)
    - [存储机制:文件存储](#%E5%AD%98%E5%82%A8%E6%9C%BA%E5%88%B6%E6%96%87%E4%BB%B6%E5%AD%98%E5%82%A8)
    - [存储机制:元数据存储](#%E5%AD%98%E5%82%A8%E6%9C%BA%E5%88%B6%E5%85%83%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8)
  - [HDFS高可用(HA)](#hdfs%E9%AB%98%E5%8F%AF%E7%94%A8ha)
  - [安全模式](#%E5%AE%89%E5%85%A8%E6%A8%A1%E5%BC%8F)

<!-- /TOC -->
# hadoop 分布式文件系统
管理网络中跨多台计算机存储的文件系统称为分布式文件系统(distributed filesystem)
## Hadoop 概念
* HDFS 产生背景
随着数据量越来越大，在一个操作系统管辖的范围内存不下了，那么就分配到更多的操作系统管理的磁盘中，但是不方便管理和维护，迫切需要一种系统来管理多台机器上的文件，这就是分布式文件管理系统。 HDFS 只是分布式文件管理系统中的一种。
* 设计目标
    * 商用硬件，运行在大量廉价商用机器上:硬件错误是常态，提供容错机制
    * 简单一致性模型:HDFS 文件写入只支持单个写入者，一次写入多次读取，支持追加，不允许修改，保证数据一致性
    * 流式数据访问:批量读而非随机读，关注吞吐量而非时间，HDFS 的构建思路：一次写入、多次读取是最高效的访问模式。读取整个数据集的时间延迟比读取第一条记录的时间延迟更重要
    * 存储大规模数据集:典型文件大小GB~TB，关注横向线性扩展，目前已有存储PB级数据的Hadoop集群
* HDFS 概念
HDFS (Hadoop Distributed File System)
    * 文件系统，用于存储文件，通过目录树来定位文件
    * 分布式的，由很多服务器联合起来实现其功能，集群中的服务器有各自的角色
* 优点
    HDFS 的设计适合一次写入，多次读出的场景，且不支持文件的修改
    * 高容错、高可用、高扩展
        * 数据冗余多副本，副本丢失后自动恢复
        * NameNode HA、安全模式
        * 10K节点规模
    * 海量数据存储
        * 典型文件大小GB~TB， 百万以上文件数量， PB以上数据规模
    * 构建成本低、安全可靠
        * 构建在廉价的商用服务器上
        * 提供了容错和恢复机制
    * 适合大规模离线批处理
        * 流式数据访问
        * 数据位置暴露给计算框架
* 缺点
    * 不适合低延迟数据访问
    * 不适合大量小文件存储
        * 元数据占用NameNode大量内存空间
        * 磁盘寻道时间超过读取时间
    * 不支持并发写入
        * 一个文件同时只能有一个写入者
    * 不支持文件随机修改
        * 仅支持追加写入
## HDFS 组成架构
![avatar](https://github.com/zXingchu/bigdata/blob/master/hadoop/hadoop架构.png "hadoop架构")
HDFS集群有两类节点以管理节点-工作节点模式运行，即一个namenode(管理节点)和多个datanode(工作节点)。
* Active NameNode(AN)
    * 活动Master管理节点(集群中唯一)
    * 管理命名空间
    * 管理元数据:文件的位置、所有者、权限、数据块等，维护文件系统树及整棵树内所有文件和目录。这些信息以两个文件的形式永久保存在本地磁盘上，命名空间镜像文件和编辑日志文件。
    * 管理Block副本策略:默认3个副本
    * 处理客户端读写请求，为DataNode分配任务
    * 两种容错机制:备份那些组成文件系统元数据持久状态的文件；运行一个辅助namenode
* Standby NameNode(SN)
    * 热备Master管理节点(Active NameNode的热备节点)
        * Hadoop 3.0允许配置多个Standby NameNode
    * Active NameNode宕机后，快速升级为新的Active
    * 定期合并编辑日志与命名空间镜像:周期性同步edits编辑日志，定期合并fsimage与edits到本地磁盘
* NameNode元数据文件
    * edits(编辑日志文件) :保存了自最新检查点(Checkpoint)之后的所有文件更新操作
    * fsimage(元数据检查点镜像文件):保存了文件系统中所有的目录和文件信息，如:某个目录下有哪些子目录和文件，以及文件名、文件副本数、文件由哪些Block组成等
    * Active NameNode内存中有一份最新的元数据(= fsimage + edits)
    * Standby NameNode在检查点定期将内存中的元数据保存到fsimage文件中
* 联邦HDFS
    * namenode在内存中保存文件系统中每个文件和每个数据块的引用关系，内存称为限制系统横向扩展的瓶颈。
    * 2.x发行版本系列中引入的联邦HDFS允许系统通过添加namenode实现扩展，其中每个namenode管理文件系统命名空间中的一部分。
    * 联邦环境下，每个namenode维护一个命名空间卷(namespace volume)，由命名空间的元数据和一个数据块池(block pool)组成，数据池包含该命名空间下文件的所有数据块。命名空间卷之间相互独立。
    * 集群中的datanode需要注册到每个namenode，并且存储着来自多个数据块池中的数据块
    * 客户端需要使用客户端挂载数据表将文件路径映射到namenode
* DataNode
    * Slave工作节点(可大规模扩展)
    * 存储Block和数据校验和
    * 执行客户端发送的读写操作
    * 通过心跳机制定期(默认3秒)向NameNode汇报运行状态和Block列表信息
    * 集群启动时， DataNode向NameNode提供Block列表信息
    * 块缓存:对于访问频繁的文件，对应的块可能被显式缓存在datanode的内存中，以堆外缓存(off-head block cache)的形式存在(默认一个块仅缓存在一个datanode内存中)。用户或应用通过在缓存池中增加一个cache directive告诉namenode需要缓存哪些文件及存多久。缓存池是一个用于管理缓存权限和资源使用的管理性分组。
* Block数据块
    * HDFS最小存储单元
    * 文件写入HDFS会被切分成若干个Block
    * Block大小固定， 默认为128MB，可自定义
    * 若一个Block的大小小于设定值，不会占用整个块空间
    * 默认情况下每个Block有3个副本
* Client
    * 将文件切分为Block
    * 与NameNode交互，获取文件元数据
    * 管理HDFS
## HDFS 存储机制
### 存储机制:文件存储
![avatar](https://github.com/zXingchu/bigdata/blob/master/hadoop/hadoop文件块DataNode.png "hadoop DataNode文件块")
* Block存储
    * Block是HDFS的最小存储单元
    * 如何设置Block大小
        * 目标:最小化寻址开销，降到1%以下
        * 默认大小: 128M
        * 块太小:寻址时间占比过高
        * 块太大: Map任务数太少，作业执行速度变慢
    * Block和元数据分开存储: Block存储于DataNode，元数据存储于NameNode
    * Block多副本
        * 以DataNode节点为备份对象
        * 机架感知:将副本存储到不同的机架上，实现数据的高容错
        * 副本均匀分布:提高访问带宽和读取性能，实现负载均衡
* Block副本放置策略
    * 副本1:放在Client所在节点
        * 对于远程Client，系统会随机选择节点
    * 副本2:放在不同的机架节点上
    * 副本3:放在与第二个副本同一机架的不同节点上
    * 副本N: 随机选择
    * 节点选择: 同等条件下优先选择空闲节点
* Block文件
    * Block文件是DataNode本地磁盘中名为“blk_blockId” 的Linux文件
        * DataNode在启动时自动创建存储目录，无需格式化
        * DataNode的current目录下的文件名都以 'blk\_' 为前缀
        * Block元数据文件(\*.meta)由一个包含版本、类型信息的头文件和一系列校验值组成
### 存储机制:元数据存储
![avatar](https://github.com/zXingchu/bigdata/blob/master/hadoop/hadoop文件块NameNode.png "hadoop NameNode文件块")
* 元数据的两种存储形式
    * 内存元数据(NameNode)
    * 文件元数据(edits + fsimage)
* edits(编辑日志文件)
    * Client请求变更操作时，操作首先被写入edits，再写入内存
    * edits文件名通过前/后缀记录当前操作的Transaction Id
* fsimage(元数据镜像检查点文件)
    * 不会为文件系统的每个更新操作进行持久化，因为写fsimage的速度非常慢
    * fsimage文件名会标记对应的Transaction Id
* edits与fsimage的合并机制
    ![avatar](https://github.com/zXingchu/bigdata/blob/master/hadoop/edits与fsimage合并.jpg "edits与fsimage合并机制")
## HDFS高可用(HA)
* 通过联合使用在多个文件系统中备份namenode的元数据和通过备用namenode创建监测点能防止数据丢失，但依旧无法实现文件系统的高可用性。namenode依旧存在单点失效(SPOF,single point of failure)的问题。namenode是唯一存储元数据与文件到数据块映射的地方，namenode失效了，那么所有的客户端，包括MapReduce作业，均无法读、写或列举文件，hadoop系统将无法提供服务直到新的namenode上线
* 从一个失效的namenode恢复，系统管理员得启动一个拥有文件系统元数据副本的新的namenode，并配置datanode和客户端。新的namenode知道满足以下情形才能响应服务:
    * 将命名空间的映像导入内存中
    * 重演编辑日志
    * 接收到足够多的来自datanode的数据块报告并退出安全模式
    * 对于一个大型集群，namenode的冷启动需要30分钟，甚至更长时间
* hadoop2对高可用性的支持:配置一堆活动-备用(active-standby)namenode:
    * namenode之间通过高可用共享存储实现编辑日志的共享
        * 两种可选高可用性共享存储:NFS过滤器或群体日志管理器(QJM,quorum journal manager)
        * QJM是一个专用的HDFS实现，为提供一个高可用的编辑日志而设计，被推荐用于大多数HDFS部署中
        * QJM以一组日志节点(journalnode)的形式运行，每一次编辑必须写入多数日志节点
    * datanode需要同时向两个namenode发送数据块处理报告
    * 客户端需要使用特定的机制来处理namenode的失效问题，这一机制对用户透明
    * 辅助namenode的角色被备用namenode所包含，备用namenode为活动的namenode命名空间设置周期性检查点
    * 活动namenode失效后，备用namenode能够快速(几十秒)实现任务接管。实际观察失效时间略长(1分钟左右)
    * 活动namenode和namenode均失效，管理员依旧可以声明一个备用namenode并实现冷启动
* Active NN与Standby NN的主备切换
* 故障转移控制器(failover controller):系统中称为故障转移控制器的新实体，管理将活动namenode转移为备用namenode的转换过程
  * 存在多种故障转移控制器
  * 默认使用ZooKeeper来确保有且只有一个活动namenode
  * 每一个namenode运行着一个轻量级的故障转移控制器，其工作为监视宿主namenode是否失效(通过简单的心跳机制实现)并在namenode失效时进行故障切换
  * 平稳的故障转移(graceful failover):管理员手动发起故障转移
  * 规避(fencing):非平稳故障转移，无法确切知道失效namenode是否已经停止运行，确保先前活动的namenode不会执行危害系统并导致系统崩溃的操作
      * QJM仅运行一个namenode向编辑日志中写入数据，但先前活动的namenode仍有可能响应并处理客户过时的读请求，因此设置一个SSH规避命令用于杀死namenode进程是一个好主意
      * 使用NFS过滤器实现共享编辑日志，无法同一时间只允许一个namenode写入，需要更有力的规避方法
          * 撤销namenode的访问共享存储目录的权限(通常使用供应商指定的NFS命令)
          * 通过远程管理命令屏蔽相应的网络端口
      * 最后手段:一枪爆头(STONITH),主要通过一个特定的供电单元对相应主机进行断电操作
* 客户端的故障转移通过客户端类库实现透明处理
    * 最简单的实现:通过客户端的配置文件实现故障转移的控制，HDFS URI使用一个逻辑主机名，改主机名映射到一对namenode地址(在配置文件中设置)，客户端类库会访问每一个namenode地址直至处理完成
* 利用QJM实现元数据高可用
    * QJM机制(Quorum Journal Manager)
        * 只要保证Quorum(法定人数)数量的操作成功，就认为这是一次最终成功的操作
    * QJM共享存储系统
        * 部署奇数(2N+1)个JournalNode
        * JournalNode负责存储edits编辑日志
        * 写edits的时候，只要超过半数(N+1)的JournalNode返回成功，就代表本次写入成功
        * 最多可容忍N个JournalNode宕机
        * 基于Paxos算法实现
* 利用ZooKeeper实现Active节点选举
    ![avatar](https://github.com/zXingchu/bigdata/blob/master/hadoop/zookeeper节点选举.jpg "zookeeper active节点选举")
## 安全模式
* 什么是安全模式
    * 安全模式是HDFS的一种特殊状态，在这种状态下， HDFS只接收读数据请求， 而不接收写入、删除、修改等变更请求
    * 安全模式是HDFS确保Block数据安全的一种保护机制
    * Active NameNode启动时， HDFS会进入安全模式， DataNode主动向NameNode汇报可用Block列表等信息，在系统达到安全标准前， HDFS一直处于“只读”状态
* 何时正常离开安全模式
    * Block上报率: DataNode上报的可用Block个数 / NameNode元数据记录的Block个数
    * 当Block上报率 >= 阈值时， HDFS才能离开安全模式，默认阈值为0.999
    * 不建议手动强制退出安全模式
* 触发安全模式的原因
    * NameNode重启
    * NameNode磁盘空间不足
    * Block上报率低于阈值
    * DataNode无法正常启动
    * 日志中出现严重异常
    * 用户操作不当，如:强制关机(特别注意！)
* 故障排查
    * 找到DataNode不能正常启动的原因， 重启DataNode
    * 清理NameNode磁盘
    * 谨慎操作，有问题找星环，以免丢失数据
