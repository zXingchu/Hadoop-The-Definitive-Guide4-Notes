<!-- TOC -->

- [YARN](#yarn)
    - [YARN 应用运行机制](#yarn-应用运行机制)
        - [资源请求](#资源请求)
        - [应用生命期](#应用生命期)
        - [构建 YARN 应用](#构建-yarn-应用)
    - [YARN 与 MapReduce 1 相比](#yarn-与-mapreduce-1-相比)
    - [YARN 中的调度](#yarn-中的调度)
        - [调度选项](#调度选项)
        - [容量调度器配置](#容量调度器配置)
            - [队列放置](#队列放置)
        - [公平调度器配置](#公平调度器配置)
            - [启用公平调度器](#启用公平调度器)
            - [队列配置](#队列配置)
            - [队列放置](#队列放置-1)
            - [抢占](#抢占)
        - [延迟调度](#延迟调度)
        - [主导资源公平性](#主导资源公平性)

<!-- /TOC -->
# YARN
Apache YARN(Yet Another Resource Negotiator)是Hadoop的集群资源管理系统。具有足够的通用性，同样可以支持其他的分布式计算
YARN提供请求和使用集群资源的API。用户代码中往往使用的是分布式计算框架提供的更高层API，这些API建立在YARN之上且向用户隐藏了资源管理细节
![avatar](E:/学习/bigdata/yarn/YARN应用.png "YARN应用")
如上图，一些分布式计算框架(MapReduce，Spark等等)作为YARN应用运行在集群计算层(YARN)和集群存储层(HDFS和HBase)上
而Pig，Hive和Crunch都是运行在MapReduce，Spark或Tez之上的处理框架，它们不和YARN直接交互
## YARN 应用运行机制
YARN通过两类长期运行的守护程序提供自己的核心服务：
* 管理集群上资源使用的资源管理器(resource manager)
* 运行在集群所有节点上且能够启动和监控容器(container)的节点管理器(node manager)
    * 容器用于执行特定应用程序的进程，每个容器都有资源限制(内存、CPU等)
    * 一个容器可以是一个Unix进程，也可以是一个Linux cgroup，取决于YARN的配置

![avatar](E:/学习/bigdata/yarn/YARN应用的运行机制.png "YARN应用的运行机制")

在YARN上运行一个应用：
* 步骤一：客户端联系资源管理器，要求它运行一个application master进程
* 步骤二：资源管理器找到一个能够在容器中启动application master的节点管理器
* 步骤三：application master运行后能做什么依赖于应用本身
    * 可能简单地运行一个计算，并返回结果给客户端
    * 可能向资源管理器请求更多的容器
* 步骤四：运行更多的容器以用于分布式计算

可以看出，YARN本身不会为应用的各部分(客户端、master和进程)彼此间通信提供任何手段。大多数YARN应用使用某种形式的远程通信机制(例如Hadoop的RPC层)来向客户端传递状态更新和返回结果，这些通信机制都是专属于各应用的
### 资源请求
YARN有灵活的资源请求模型。当请求多个容器时，可以指定每个容器需要的计算机资源数量(内存和CPU)，还可以指定对容器的本地限制要求
YARN允许一个应用为所申请的容器指定本地限制。本地限制可用于申请位于指定节点或机架，或集群中任何位置的容器。本地限制无法满足，要么不分配资源，要么选择放松限制。节点->同一机架节点->集群节点
YARN应用可以在运行中的任意时刻提出资源申请。最开始提出所有申请或动态地在需要更多资源时提出申请。Spark采用第一种方式，在集群上启动固定数量的执行器。MapReduce分两步走
### 应用生命期
YARN应用的生命期差异性很大：短期应用和长期应用
按照应用到用户运行的作业之间的映射关系对应进行分类。
* 最简单的模型：一个用户作业对应一个应用
    * MapReduce
* 第二种模型：作业的每个工作流或每个用户对话(可能并无关联性)对应一个应用
    * 效率更高，容器作业间重用，可能缓冲作业间中间数据
    * Spark
* 第三种模型：多个用户共享一个长期运行的应用
    * 通常作为一种协调者的角色在运行。
    * Apache Slider，Impala。有一个长期运行的application master，避免启动新的application master带来的开销，一个总是启动的application master意味着用户将获得非常低延迟的查询响应

### 构建 YARN 应用
从无到有编写一个YARN应用相当复杂，可以直接使用现成的应用
Spark、Tez：图
Spark、Samza、Storm：流处理
Apache Slider：简化构建YARN应用的过程。使在YARN上运行现有的分布式应用成为可能。Slide提供控制手段，可以修改应用运行所在的节点的数量，也可以暂停和恢复应用的运行
Apache Twill：与Slider类似，额外提供一个简单的编程模型，用于开发YARN分布式应用。Twill运行将集群进程定义为Java Runnable的扩展，然后在集群上的YARN容器中运行它们。同样为实时日志和命令消息提供支持
YARN项目的一部分的distributed shell应用为如何写YARN应用做了示范。演示了如何使用YARN客户端API来处理客户端或application master与YARN守护进程之间的通信
## YARN 与 MapReduce 1 相比
| MapReduce 1 | YARN |
| ------ | ------ |
| Jobtracker | 资源管理器、application master、时间轴服务器 |
| Tasktracker | 节点管理器 |
| Slot | 容器 |
YARN的很多设计是为了解决MapReduce 1的局限性
* 可扩展性
    * YARN可以在更大规模的集群上运行
    * 节点数达到4000，任务数达到40000，MapReduce 1会遇到可扩展瓶颈，源于jobtracker必须同时管理作业和任务
    * YARN你用资源管理器和application master分离的架构，可扩展到近10000个节点和100000个任务
* 可用性
    * 当服务守护进程失败，通过为另一个守护进程复制接管工作，从而获得高可用性(HA,high availability)
    * jobtracker内存中大量快速变化的复杂状态使改进其服务获得HA非常困难
    * YARN中jobtracker在资源管理器和application master之间进行职责划分，HA成为分而治之的问题：
        * 先为资源管理器提供HA，再为YARN应用提供HA
* 利用率
    * MapReduce 1中，每个tasktracker都配置若干固定长度的slot，静态分配，划分为map slot和reduce slot和任务一对一
    * YARN中，一个节点管理器管理一个资源池，获得运行任务的资源，应用就会正常进行(MapReduce 1仅有map slot导致等待)
    * YARN中的资源精细化管理，应用按需请求资源
* 多租户
    * YARN的一大优点在于向MapReduce以外的其他类型的分布式应用开放了Hadoop

## YARN 中的调度
YARN调度器的工作就是根据既定策略为应用分配资源
### 调度选项
YARN中有三种调度器可用：
* FIFO调度器(FIFO Scheduler)
    * 简单易懂，无需配置，不适合共享集群。
    * 大应用占用资源
* 容量调度器(Capacity Scheduler)
    * 一个独立的专门队列保证小作业一提交就可以启动
    * 以整个集群的利用率为代价
* 公平调度器(Fair Scheduler)
    * 不需要预留一定量的资源，调度器在所有运行的作业间动态平衡
    * 第二个作业启动到获得公平共享资源之间有时间延迟，等待第一个作业使用的容器用完并释放出资源

![avatar](E:/学习/bigdata/yarn/不同调度运行大小作业.png "不同调度运行大小作业")
### 容量调度器配置
容量调度器允许多个组织共享一个Hadoop集群，每个组织可以分配到全部集群资源的一部分。每个组织被配置一个专门的队列，每个队列被配置为可使用一定的集群资源。队列可以进一步按层次划分。在队列中使用FIFO调度
* 弹性队列：单个作业使用的资源不会超过队列容量。队列中多个作业，队列资源不足，则容量调度器可能会将空余的资源分配给队列中的作业，即使超过队列容量
* 最大容量限制：防止队列过多侵占其他队列的容量，牺牲队列弹性
假设一个队列的层次结构如下：
* root
    * prod
    * dev
        * eng
        * science

基于上述队列层次的容量调度器配置文件如下，文件名为capacity-scheduler.xml。对特定队列进行配置，是通过以下形式的配置属性yarn.scheduler.capacity.<queue-path>.<sub-property>进行设置，其中<queue-path>表示队列的层次路径
范例-容量调度器的基本配置文件
```
<?xml version="1.0"?>
<configuration>
  <property>
    <name>yarn.scheduler.capacity.root.queues</name>
    <value>prod,dev</value>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.dev.queues</name>
    <value>eng,science</value>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.prod.capacity</name>
    <value>40</value>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.dev.capacity</name>
    <value>60</value>
  </property>
  !--- maximum-capacity ---!
  <property>
    <name>yarn.scheduler.capacity.root.dev.maximum-capacity</name>
    <value>75</value>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.dev.eng.capacity</name>
    <value>50</value>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.dev.science.capacity</name>
    <value>50</value>
  </property>
</configuration>
```
除配置队列层次和容量，还有设置用来控制
* 单个用户或应用能被分配到的最大资源数量
* 同时运行的应用数量
* 队列的ACL认证
* ...

更多内容参见 [capacity_schedular](http://bit_ly/capacity_schedular "capacity_schedular")
#### 队列放置
* 将应用放置在哪个队列，取决于应用本身
    * MapReduce中，可通过设置属性mapreduce.job.queuename指定队列
    * 队列不存在提交时将发送错误
    * 不指定队列则放在“default”的默认队列中
* 队列名应该是队列层次名的最后一部分
    * 对上述配置范例，prod和eng为合法的队列名，root.dev.eng作为队列名无效

### 公平调度器配置
公平调度器旨在为所有运行的应用公平分配资源。下图显示了同一队列实现资源公平共享，公平共享也可在多个队列间工作
![avatar](E:/学习/bigdata/yarn/用户队列间的公平共享.png "用户队列间的公平共享")
#### 启用公平调度器
* 公平调度器的使用由属性yarn.resourcemanager.scheduler.class的设置决定
    * 默认使用容量调度器
        * 一些Hadoop分布式项目，如CDH中默认公平调度器
* 使用公平调度器，需将yarn-site.xml文件中yarn.resourcemanager.scheduler.class设置为公平调度器的完全限定名
    * 公平调度器完全限定名：org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler

#### 队列配置
* 通过名为fair-scheduler.xml的分配文件对公平调度器进行配置
    * 文件位于类路径下(可通过设置属性yarn.scheduler.fair.allocation.file修改文件名)
    * 无分配文件，调度器工作策略：每个应用放置在一个以用户名命名的队列中，队列在用户提交第一个应用时动态创建
    * 分配文件可以为每个队列进行配置。可以对容量调度器支持的层次队列进行配置

范例-公平调度器的分配文件
```
<?xml version="1.0"?>
<allocations>
  <defaultQueueSchedulingPolicy>fair</defaultQueueSchedulingPolicy>

  <queue name="prod">
    <weight>40</weight>
    <schedulingPolicy>fifo</schedulingPolicy>
  </queue>

  <queue name="dev">
    <weight>60</weight>
    <queue name="eng" />
    <queue name="science" />
  </queue>

  <queuePlacementPolicy>
    <rule name="specified" create="false" />
    <rule name="primaryGroup" create="false" />
    <rule name="default" queue="dev.eng" />
  </queuePlacementPolicy>
</allocations>
```
* 设置权重时，须考虑默认队列和动态创建的队列。虽然没有指定权重，但仍有值为1的权重
* 队列的默认调度策略通过顶层元素defaultQueueSchedulingPolicy进行设置，默认公平调度
    * 公平调度器也支持队列级别的FIFO策略，Dominant Resource Fairness(drf)策略
* 队列调度策略可通过该队列schedulingPolicy元素指定的策略覆盖
    * 如上述配置，prod队列使用FIFO调度策略，队列之间仍使用公平调度
* 每个队列可配置最大和最小资源数量，最大可运行的应用的数量
    * 最小资源数量不是一个硬性限制，调度器常用它对资源分配进行优先排序，也用于抢占行为

#### 队列放置
* 公平调度器使用一个基于规则的系统来确定应用应该放置到哪个队列
* 如上范例queuePlacementPolicy元素包含一个规则列表，规则被依次尝试，知道匹配成功
    * specified，表示把应用放进所指明的队列中
    * primaryGroup，试着把应用放在以用户的主Unix组名命名的队列中，无此队列，尝试下一条而不是创建
    * default，兜底规则，前述规则不匹配，启用该规则
* 完全省略queuePlacementPolicy元素，此时队列放置默认遵从如下规则
    * 即除非明确定义队列，否则必要时会以用户名为队列名创建队列
```
<queuePlacementPolicy>
    <rule name="specified" />
    <rule name="user" />
</queuePlacementPolicy>
```
* 简单的队列放置策略：将所有的应用放进同一个队列，在应用之间公平共享资源，而不是在用户之间共享，等价于如下规则：
    * 不适用分配文件设置以上策略，将属性yarn.scheduler.fair.user-as-default-queue设为false
    * 将属性yarn.scheduler.fair.allow-undeclared-pools设为false，用户无法随意创建队列
```
<queuePlacementPolicy>
    <rule name="default" />
</queuePlacementPolicy>
```
#### 抢占
* 为了使作业从提交到执行所需的时间可预测，公平调度器支持“抢占”功能
  * 抢占，即允许调度器终止那些占用资源超过了其公平共享份额的队列的容器
  * 抢占会降低集群效率，被终止的containers需要重新执行
* 通过将yarn.scheduler.fair.preemption设置为true，可全面启用抢占功能
  * 两个相关的抢占超时设置，设定时间均为秒级，默认情况无设置，为了允许抢占，需要至少设置一个超时参数
      * 用于最小共享(minimum share preemption timeout)
      * 用于公平共享(fair share preemption timeout)
* 队列在minimum share preemption timeout指定的时间内未获得被承诺的最小共享资源，调度器将抢占其他容器
    * 可通过分配文件中顶层元素defaultMinSharePreemptionTimeout为所有队列设置默认超时时间
    * 可通过设置每个队列的minSharePreemptionTimeout元素为单个队列指定超时时间
* 队列在fair share preemption timeout指定时间内获得的资源仍然低于其公平共享资源份额的一半，调度器将抢占其他容器
    * 可通过分配文件中顶层元素defaultFairSharePreemptionTimeout为所有队列设置默认超时时间
    * 可通过设置每个队列的fairSharePreemptionTimeout元素为单个队列指定超时时间
    * 可通过设置fairSharePreemptionThreshold和fairSharePreemptionThreshold(每个队列)修改超时阈值，默认值0.5

### 延迟调度
* 延迟调度：所有的YARN调度器都试图以本地请求为重。在一个繁忙的集群上，应用请求的节点极有可能有其他容器正在该节点上运行。可以放宽本地性需求。但通过实践，此时如果等待一小段实践，能够喜剧性的增加在所请求的节点上分配到一个容器的机会，从而提高集群的效率
    * 容量调度器和公平调度器都支持延迟调度
* YARN中的每个节点管理器周期性的(默认一秒一次)向资源管理器发送心跳请求
    * 心跳中携带节点管理器中正运行的容器、新容器可用的资源等信息
    * 对于计划运行一个容器的应用而言，每个心跳就是一个潜在的调度机会(scheduling opportunity)
* 使用延迟调度时，调度器不会简单的使用它收到的第一个调度机会，而是等待设定的最大数目的调度机会发生，然后才放松本地性限制并接收笑一个调度机会
* 容量调度器，可通过yarn.scheduler.capacity.node-locality-delay配置延迟调度
    * 设置为正整数，表示调度器在放松节点限制前准备错过的调度机会的数量
* 公平调度器也使用调度机会的数量决定延迟时间，使用集群规模的比例来表示这个值
    * yarn.scheduler.fair.locality.threshold.node
        * 设置值为0.5表示调度器在接受同一机架中的其他节点之间，将一直等待直到集群中的一半节点都给过调度机会
    * yarn.scheduler.fair.locality.threshold.rack
        * 表示在接受另一个机架替代所申请的机架之前需要等待的时长阈值

### 主导资源公平性
* 对于多种资源类型需要调度，YARN中调度器观察每个用户的主导资源，并将其作为对集群资源使用的一个度量，称为“主导资源公平性”(Dominant Resource Fairness,DRF)
* 默认情况不使用DRF，在资源计算期间只考虑内存
* 容量调度器进行配置后，将capacity-scheduler.xml文件中的org.apache.hadoop.yarn.util.resource.DominantResourceCalculator设为yarn.scheduler.capacity.resource-calcalator可以使用DRF
* 公平调度器通过将分配文件中的顶层元素defaultQueueSchedulingPolicy设为drf可以使用DRF

