## Hadoop 概念
* HDFS 产生背景
随着数据量越来越大，在一个操作系统管辖的范围内存不下了，那么就分配到更多的操作系统管理的磁盘中，但是不方便管理和维护，迫切需要一种系统来管理多台机器上的文件，这就是分布式文件管理系统。 HDFS 只是分布式文件管理系统中的一种。
* 设计目标
    * 运行在大量廉价商用机器上:硬件错误是常态，提供容错机制
    * 简单一致性模型:一次写入多次读取，支持追加，不允许修改，保证数据一致性
    * 流式数据访问:批量读而非随机读，关注吞吐量而非时间
    * 存储大规模数据集:典型文件大小GB~TB，关注横向线性扩展
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
![avatar](E:/学习/bigdata/hadoop/hadoop架构.png "hadoop架构")
* Active NameNode(AN)
    * 活动Master管理节点(集群中唯一)
    * 管理命名空间
    * 管理元数据:文件的位置、所有者、权限、数据块等
    * 管理Block副本策略:默认3个副本
    * 处理客户端读写请求，为DataNode分配任务
* Standby NameNode(SN)
    * 热备Master管理节点(Active NameNode的热备节点)
        * Hadoop 3.0允许配置多个Standby NameNode
    * Active NameNode宕机后，快速升级为新的Active
    * 周期性同步edits编辑日志，定期合并fsimage与edits到本地磁盘
* NameNode元数据文件
    * edits(编辑日志文件) :保存了自最新检查点(Checkpoint)之后的所有文件更新操作
    * fsimage(元数据检查点镜像文件):保存了文件系统中所有的目录和文件信息，如:某个目录下有哪些子目录和文件，以及文件名、文件副本数、文件由哪些Block组成等
    * Active NameNode内存中有一份最新的元数据(= fsimage + edits)
    * Standby NameNode在检查点定期将内存中的元数据保存到fsimage文件中
* DataNode
    * Slave工作节点(可大规模扩展)
    * 存储Block和数据校验和
    * 执行客户端发送的读写操作
    * 通过心跳机制定期(默认3秒)向NameNode汇报运行状态和Block列表信息
    * 集群启动时， DataNode向NameNode提供Block列表信息
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
![avatar](E:/学习/bigdata/hadoop/hadoop文件块DataNode.png "hadoop DataNode文件块")
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
![avatar](E:/学习/bigdata/hadoop/hadoop文件块NameNode.png "hadoop NameNode文件块")
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
    ![avatar](E:/学习/bigdata/hadoop/edits与fsimage合并.jpg "edits与fsimage合并机制")
## 读写操作
### 读写操作:写操作
![avatar](E:/学习/bigdata/hadoop/写操作.png "写操作")
### 读写操作:读操作
![avatar](E:/学习/bigdata/hadoop/读操作.png "读操作")
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
## HDFS高可用
* Active NN与Standby NN的主备切换
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
    ![avatar](E:/学习/bigdata/hadoop/zookeeper节点选举.jpg "zookeeper active节点选举")
