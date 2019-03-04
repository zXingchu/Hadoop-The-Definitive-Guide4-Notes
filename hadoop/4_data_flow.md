## 数据流
### 文件读取
![avatar](E:/学习/bigdata/hadoop/客户端读取HDFS中的数据.png "客户端读取HDFS中的数据")
* 步骤一:客户端通过调用FileSystem对象的open()方法来打开希望读取的文件，对于HDFS来说，这个对象是DistributedFileSystem的一个实例
* 步骤二:DistributedFileSystem通过远程调用(RPC)来调用namenode，以确定文件起始块的位置
    * 对于每一个块，namenode返回存有该块副本的datanode地址，datanode根据与客户端的距离排序
    * DistributedFileSystem类返回一个FSDataInputStream对象(一个支持文件定位的输入流)给客户端读取数据
    * FSDataInputStream转而封装DFSInputStream对象，该对象管理datanode和namenode的I/O
* 步骤三:客户端对该输入流调用read()方法
* 步骤四:存储文件起始几个块的datanode地址的DFSInputStream随即连接距离最近的文件中第一个块所在的datanode。通过对数据流反复调用read()方法，将数据从datanode传输到客户端
* 步骤五:到达块的末端时，DFSInputStream关闭与该datanode的连接，然后寻找下一个块的最佳datanode
    * 客户端从流中读取数据时，块是按照打开DFSInputStream与datanode新建连接的顺序读取的，它也会根据需要询问namenode来检索下一批数据块的datanode位置
* 步骤六:一旦客户端完成读取，就对FSDataInputStream调用close()方法
    * 读取数据时，如果DFSInputStream在与datanode通信时遇到错误，会尝试从另一个最邻近datanode读取数据，并且记住故障datanode，以保证不会反复读取该节点上的后续的块
    * DFSInputStream会通过校验和确认从datanode发来的数据是否完整
    * DFSInputStream发现有损坏的块，会试图从其他datanode读取复本，并将被损坏的块通知namenode
    * 设计重点:客户端可以直接连接到datanode检索数据，使HDFS扩展到大量并发客户端，namenode只需要响应块位置的请求，避免namenode称为瓶颈
#### 网络拓扑与Hadoop
彼此近邻？很难衡量节点之间的带宽，hadoop采用了一个简单的方法:把网络看成一棵树，两个节点间的距离是它们到最近共同祖先的距离总和。该树中的层次是没有预先设定的，但相对于数据中心、机架和运行节点，通常可以设定等级。具体想法是针对以下每个场景，可用带宽依次递减:
* 同一节点上的进程
* 同一机架上的不同节点
* 同一数据中心中不同机架上的节点
* 不同数据中心中的节点(目前Hadoop不适合跨数据中心运行)
假设有数据中心的d1机架r1中的节点n1。该节点可以表示为/d1/r1/n1，以下给出四种距离描述:
* distance(/d1/r1/n1, /d1/r1/n1)=0(同一节点上的进程)
* distance(/d1/r1/n1, /d1/r1/n2)=2(同一机架上的不同节点)
* distance(/d1/r1/n1, /d1/r2/n3)=4(同一数据中心中不同机架上的节点)
* distance(/d1/r1/n1, /d2/r3/n4)=6(不同数据中心中的节点)
Hadoop无法自动发现网络拓扑结构，需要配置。默认情况下，假设网络是扁平化的只有一层，即所有节点在同一数据中心的同一机架上，不需要进一步配置
### 文件写入
![avatar](E:/学习/bigdata/hadoop/客户端将数据写入HDFS.png "客户端将数据写入HDFS")
考虑如何新建一个文件，把数据写入该文件，最后关闭该文件。如上图所示
* 步骤一:客户端通过对DistributedFileSystem对象调用create()来新建文件
* 步骤二:DistributedFileSystem对namenode创建一个RPC调用，在文件系统的命名空间中新建一个文件，此时文件无数据块
    * namenode执行各种不同的检查确保该文件不存在且客户端有权限新建该文件。
    * 检查通过则为创建新文件记录一条记录
    * 检查不通过则创建失败并抛出IOException异常
    * DistributedFileSystem向客户端返回一个FSDataOutputStream对象，由此客户端可以开始写入数据
    * 就像读取事件一样，FSDataOutputStream封装一个DFSOutPutStream对象，负责处理datanode和namenode的通信
* 步骤三:客户端写入数据时，DFSOutPutStream将它分成一个个数据包，并写入内部队列(数据队列data queue)
* 步骤四:DataStreamer处理数据队列，负责挑选出适合存储数复本的一组datanode，并据此要求namenode分配新的数据块
    * 这一组datanode构成一个管线
    * 假设复本数为3，则管线内有三个节点
    * DataStreamer将数据包流式传输到管线中第一个datanode节点，该datanode存储数据包并再发送到管线中第二个datanode，以此类推
* 步骤五:DFSOutPutStream也维护着一个内部数据包队列来等待datanode的收到确认回执(确认队列ack queue)
    * 收到管道中所有datanode确认信息后，该数据包才会从确认队列删除
    * 如果任何datanode在数据写入期间发生故障，则执行以下操作:
        * 首先关闭管线，确认把队列中所有数据包都添加回数据队列的最前端，确保故障节点下游datanode不会漏掉数据包
        * 为存储在另一正常datanode的当前数据块指定一个新的标识，并将标识传送给namenode，以便故障datanode恢复后删除存储的部分数据块
        * 从管线中删除故障datanode，基于正常datanode构建新管线。余下数据块写入管线中正常的datanode
        * namenode注意到块复本量不足时，会在另一节点上创建一个新的复本
        * 后续数据块继续正常接受处理
        * 可能有多个datanode同时发生故障(非常少见)
        * 只要写入了dfs.namenode.replication.min(默认为1)的复本数，写操作就会成功，这个块可以在集群中异步复制，直到达到其目标复本数(dfs.replication默认值为3）
* 步骤六:客户端完成数据的写入后，对数据流调用close()方法
* 步骤七:步骤六操作将剩余的所有数据包写入datanode管线，并在联系到namenode告知其文件写入完成之前，等待确认
    * namenode已经知道文件由哪些块组成(因为DataStreamer请求分配数据块)，所以它在返回成功前只需要等待数据块进行最小量的复制
复本怎么放？
* 对可靠性、写入带宽和读取带宽进行权衡
* Block副本放置策略
    * 副本1:放在Client所在节点
        * 对于远程Client，系统会随机选择节点
    * 副本2:放在不同的机架节点上
    * 副本3:放在与第二个副本同一机架的不同节点上
    * 副本N: 随机选择
    * 节点选择: 同等条件下优先选择空闲节点
### 一致模型
文件系统中的一致模型(coherency model)描述了文件读/写的数据可见性。HDFS牺牲了一些POSIX(Portable Operating System Interface of UNIX)要求，因此一些操作与期望的可能不同
新建一个文件后，它能在文件系统的命名空间中立即可见，如下所示
```
Path p = new Path("p");
Fs.create(p);
assertThat(fs.exists(p),is(true));
```
但写入文件的内容并不保证立即可见，即使数据流已经刷新并存储。所以文件长度显示为0:
```
Path p = new Path("p");
OutPutStream out = fs.create(P);
out.write("content".getBytes("UTF-8"));
out.flush();
assertThat(fs.getFileStatus(p).getLen(), is(0L));
```
当写入的数据超过一个块后，第一个数据库对新的reader就是可见的。之后的块也不例外。总之，当前正在写入的块对其他reader不可见
HDFS提供了一种强行将所有缓存刷新到datanode中的手段，即对FSDataOutputStream调用hflush()方法。
当hflush()方法返回成功，对所有reader，HDFS保证文件中到目前为止写入的数据均到达所有datanode的写入管道并且对所有新的reader均可见:
```
Path p = new Path("p");
OutPutStream out = fs.create(P);
out.write("content".getBytes("UTF-8"));
out.hflush();
assertThat(fs.getFileStatus(p).getLen(), is(((long) "content".length())));
```
注意，hflush()不保证datanode已经将数据写到磁盘上，仅确保数据在datanode的内存中。为确保数据写到磁盘，可用hsync()
hsync()操作类似POSIX中的fsync()系统调用，该调用提交的是一个文件描述符的缓冲数据。如，利用标准Java API数据写入本地文件，我们能够在刷新数据流且同步之后看到文件内容:
```
Path p = new Path("p");
OutPutStream out = fs.create(P);
out.write("content".getBytes("UTF-8"));
out.flush();        //flush to operating system
out.getFD().sync(); //sync to disk
assertThat(fs.getFileStatus(p).getLen(), is(((long) "content".length())));
```
在HDFS中关闭文件隐含执行hflush()方法
#### 对应用设计的重要性
如果不调用hflush()或hsync()方法，就要准备好在客户端或系统发生故障时可能会丢失数据块。需要在适当的地方调用hflush()方法。尽管hflush()操作被设计成尽量减少HDFS负载，但它有许多额外的开销(hsync()的开销更大)。
所以在数据鲁棒性和吞吐量间就会有所取舍。怎样权衡和具体的应用相关，通过度量应用程序以不同频率调用hflush()或hsync()时呈现的性能，选择一个合适的调用频率。
### 通过distcp并行复制
为了提高性能，需要写一个程序来并行处理文件。Hadoop自带一个有用程序distcp，该程序可以并行从hadoop文件系统中复制大量数据，也可以将大量数据复制到Hadoop中
#### distcp用法
Distcp的一种用法是替代hadoop fs -cp。例如，可以将文件复制到另一个文件中:
```
% hadoop distcp file1 file2
% hadoop distcp dir1 dir2 //复制目录
```
若dir2不存在，将新建dir2，目录dir1的内容全部复制到dir2下。可指定多个源路径
若dir2已存在，dir1将被复制到dir2下，形成目录结构dir2/dir1
-overwrite 选项，在保持同样的目录结构的同时强制覆盖原有文件
-update 选项，仅更新发生变化的文件。如修改了dir1子树中的文件，可以运行 % hadoop distcp -update dir1 dir2 将修改同步到dir2中
distcp的一个常见使用实例是在两个HDFS集群间传送数据。例如:
```
% hadoop distcp -update -delete -p hdfs://namenode1/foo hdfs://namenode2/foo
```
该命令在第二个集群上为第一个集群/foo目录创建了一个备份。
-delete 选项使distcp可以删除目标路径中任意没在源路径中出现的文件或目录
-p 选项意味着文件状态属性如权限、块大小和复本数被保留
如果两个集群运行的HDFS不兼容版本，可以将webhdfs协议用于它们之间的distcp:
```
% hadoop distcp -update -delete -p webhdfs://namenode1:50070/foo webhdfs://namenode2:50070/foo
```
另一个变种是使用HttpFs代理作为distcp源或目标，这样具有设置防火墙和控制带宽的优点
#### distcp原理
distcp是作为一个MapReduce作业来实现的，该复制作业是通过集群中并行运行的map来完成。每个文件通过一个map进行复制，并且distcp试图为每一个map分配大致相等的数据来执行，即把文件划分为大致相等的快。默认情况下，将近20个map被使用，可以通过 distcp -m 参数来修改map的数量


### 读写操作:写操作
![avatar](E:/学习/bigdata/hadoop/写操作.png "写操作")
### 读写操作:读操作
![avatar](E:/学习/bigdata/hadoop/读操作.png "读操作")