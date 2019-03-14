# ZooKeeper
* 介绍如何使用ZooKeeper来构建一般的分布式应用
* ZooKeeper是Hadoop的分布式协调服务
* 部分失败(partial failure)
    * 当一条消息在网络中两个节点之间传递时，如果出现网络错误，发送者无法知道接受者是否已经收到消息。接受者可能已经收到消息，也可能没有，或者接收者的进程已经死掉。发送者必须重新连接接收者，并询问以获得真实情况
* 部分失败是分布式系统固有特征
* ZooKeeper提供一组工具，使在构建分布式应用时能够对部分失败进行正确处理
* ZooKeeper具有特点：
    * 简单的
        * ZooKeeper的核心是一个精简的文件系统，提供一些简单的操作和一些额外的抽象操作，如排序和通知
    * 富有表现力的
        * ZooK的基本操作是一组丰富的“构件”(building block)，可用于实现多种协调数据结构和协议。包括：分布式队列、分布式锁和一组节点中的“领导者选举”(leader election)
    * 高可用的
        * ZooKeeper运行于一组机器之上，并且在设计上具有高可用性，因此应用程序完全可以依赖于它
        * ZooKeeper可以帮助系统避免出现单点故障，用于构建一个可靠的应用程序
    * 采用松耦合交互方式
        * 在ZooKeeper支持的交互过程中，参与者不需要彼此了解
            * 例如ZooKeeper可以被用于实现“数据汇集”(rebdezvous)，让进程在不了解其他进程(或网络状况)的情况下能够彼此发现并进行信息交互
        * 甚至可以不必同时存在，一个进程可以在ZooKeeper中留下一条消息，在该进程结束后，另一个进程还可以读取该消息
    * 是一个资源库
        * 提供了一个通用协调模式实现方法的开源共享库，使程序员免于写这类通用的协议
        * 所有人可以对这个资源库进行添加和改进
    * 高性能的
        * 在Yahoo!公司，对于写操作为主的工作负载来说，ZooKeeper的基准吞吐量已经超过每秒10000个操作。以读为主，更是高出好几倍
## 安装和运行 ZooKeeper
* 运行ZooKeeper需要Java
* 最简单的方式是在一台ZooKeeper服务器上以独立模式(standalone mode)运行
* ZooKeeper发布页面 http://hadoop.apache.org/zookeeper/releases.html
```
% tar xzf zookeeper-x.y.z.tar.gz
```
* ZooKeeper提供了几个能够运行服务并与之交互的二进制可执行文件，加入命令行路径
    ```
    % export ZOOKEEPER_HOME=~/sw/zookeeper-x.y.z
    % export PATH=$PATH:$ZOOKEEPER_HOME/bin
    ```
* 运行ZooKeeper之前，需要创建一个配置文件，习惯上命名为zoo.cfg，并保存在conf子目录中
    * 也可保存在/etc/zookeeper子目录中
    * 也可设置环境变量ZOOCFGDIR，保存在该环境变量所指定的目录中
    ```
    tickTime=2000
    dataDir=/Users/tom/zookeeper
    clientPort=2181
    ```
    * 标准的Java属性文件，定义的三个属性是以独立模式运行的ZooKeeper所需的最低要求
        * tickTime指定ZooKeeper中的基本时间单位(毫秒单位)
        * dataDir指定存储持久数据的本地文件系统位置
        * clientPort指定用于监听客户端连接的端口(通常使用2181端口)
* 启动一个本地ZooKeeper服务器
    ```
    % zkServer
    ```
* 使用nc(或telnet)发送ruok命令到监听端口，检查ZooKeeper是否正在运行
    ```
    % echo ruok | nc localhost 2181
    ```
* ZooKeeper文档 http://hadoop.apache.org/
* ZooKeeper命令：四字母组合。访问http://localhost:8080/commands 获取命令列表

| 类别 | 命令 | 描述 |
| ------ | ------ | ------ |
| 服务器状态 | ruok | 如果服务器正在运行且未处于出错状态，则输出imok |
|  | conf | 输出服务器的配置信息(根据配置文件zoo.cfg) |
|  | envi | 输出服务器的环境信息，包括ZooKeeper版本，Java版本和其他系统属性 |
|  | srvr | 输出服务器的统计信息，包括延迟统计，znode的数量和服务器运行模式(standalone,leader或follower) |
|  | stat | 输出服务器的统计信息和已连接的客户端 |
|  | srst | 重置服务器的统计信息 |
|  | isro | 显示服务器是否处于只读(ro)模式(由于网络分区)，或者读/写(rw)模式 |
| 客户端连接 | dump | 列出集合体中的所有会话和短暂znode。必须连接到leader才能够使用此命令(srvr) |
|  | cons | 列出服务器客户端的连接统计信息 |
|  | crst | 重置连接统计信息 |
| 观察 | wchs | 列出服务器上所有观察的摘要信息 |
|  | wchc | 按连接列出服务器上所有的观察(观察多影响服务器性能) |
|  | wchp | 按znode路径列出服务器上所有的观察(观察多影响服务器性能) |
| 监控 | mntr | 按Java属性格式列出服务器统计信息。适合于用作Ganglia和Nagios等监控系统的信息源 |

## 示例
* 假设有一组服务器用于为客户端提供某种服务，如何维护这组服务器的成员列表
    * 成员列表不能存储在网络中的单个节点上
    * 从服务器成员列表中删除故障服务器
* 不应是一个被动的分布式数据结构，而是一个主动的、能够在某个外部事件发生时修改数据项状态的数据结构
* 以下是如何使用ZooKeeper来实现组成员管理的
### ZooKeeper 中的组成员关系
* 理解ZooKeeper的一种方法时将其看做一个具有高可用性特征的文件系统
    * 此文件系统无文件和目录，统一使用“节点”(node)的概念，称为znode
    * znode既可以作为保存数据的容器(文件)，也可以作为保存其他znode的容器(目录)
    * 所有的znode构成了一个层次化的命名空间
* 一个自然的建立组成员列表的方式就是利用这种层次结构，组名为节点名的znode为父节点，组成员名(服务器名)为节点名的znode为子节点

### 创建组
```
//cc CreateGroup A program to create a znode representing a group in ZooKeeper

import java.io.IOException;
import java.util.concurrent.CountDownLatch;

import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.Watcher.Event.KeeperState;
import org.apache.zookeeper.ZooDefs.Ids;

public class CreateGroup implements Watcher {
  
  private static final int SESSION_TIMEOUT = 5000;
  
  private ZooKeeper zk;
  private CountDownLatch connectedSignal = new CountDownLatch(1);
  
  public void connect(String hosts) throws IOException, InterruptedException {
    zk = new ZooKeeper(hosts, SESSION_TIMEOUT, this);
    connectedSignal.await();
  }
  
  @Override
  public void process(WatchedEvent event) { // Watcher interface
    if (event.getState() == KeeperState.SyncConnected) {
      connectedSignal.countDown();
    }
  }
  
  public void create(String groupName) throws KeeperException,
      InterruptedException {
    String path = "/" + groupName;
    String createdPath = zk.create(path, null/*data*/, Ids.OPEN_ACL_UNSAFE,
        CreateMode.PERSISTENT);
    System.out.println("Created " + createdPath);
  }
  
  public void close() throws InterruptedException {
    zk.close();
  }

  public static void main(String[] args) throws Exception {
    CreateGroup createGroup = new CreateGroup();
    createGroup.connect(args[0]);
    createGroup.create(args[1]);
    createGroup.close();
  }
}
```
* 此程序创建组名为/zoo的znode
* 在main()方法执行时，创建CreateGroup的实例调用connect()方法
* connect()方法实例化了一个新的ZooKeeper类的对象
    * 此类事客户端API中的主要类，用于维护客户端和ZooKeeper服务器之间的连接
    * ZooKeeper类的构造函数有三个参数
        * ZooKeeper服务的主机地址(可指定端口)
        * 以毫秒为单位的会话超时参数
        * 一个Watcher对象的实例。Watcher对象接收来自于ZooKeeper的回调，以获得各种事件的通知
            * 此例中CreateGroup是一个Watcher，被传递给ZooKeeper构造函数
* ZooKeeper实例被创建时，会启动一个线程连接到ZooKeeper服务
    * 由于对构造函数的调用时立即返回的，因此使用新建的ZooKeeper对象前要等待其与ZooKeeper服务之间成功建立连接
    * 此使用Java的CountDownLatch类来阻止使用新建的ZooKeeper对象，直到此对象准备就绪
    * Watcher类用于获取ZooKeeper对象是否就绪的信息，在它的接口中只有一个方法：
        ```
        public void process(WatchedEvent event);
        ```
* 客户端与ZooKeeper服务建立连接后，Watcher的process()方法会被调用，参数为表示该连接的事件。
* 在接收到连接事件时，通过调用connectedSignal的countDown()方法递减其计数值，使await()方法返回
    * 连接事件以Watcher.Event.KeeperStated的枚举类型值SyncConnected表示
    * 锁存器(latch)创建时带有一个值为1的计数器，用于表示在它释放所有等待线程之前需要发生的事件数
    * 调用一次countDown()方法后，计数器值变为0，则await()方法返回
* connect()方法返回，之后执行createGroup的create()方法
    * 在此方法中使用ZooKeeper实例的create()方法来创建一个新的ZooKeeper的znode
    * 参数包括：
        * 路径(用字符串表示)
        * znode的内容(字节数组)
        * 访问控制列表(ACL)，此处使用完全开放的ACL，运行任何客户端对znode进行读/写
        * 创建的znode的类型
            * 短暂的(ephemerak)，客户端断开连接，则被ZooKeeper服务删除
            * 持久的(persistent)

```
% ecport CLASSPATH=ch21-zk/target/classes/:$ZOOKEEPER_HOME/*:$ZOOKEEPER_HOME/lib/*:$ZOOKEEPER_HOME/conf
% java createGroup localhost z00
```
### 加入组
```
//JoinGroup A program that joins a group
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.ZooDefs.Ids;

public class JoinGroup extends ConnectionWatcher {
  
  public void join(String groupName, String memberName) throws KeeperException,
      InterruptedException {
    String path = "/" + groupName + "/" + memberName;
    String createdPath = zk.create(path, null/*data*/, Ids.OPEN_ACL_UNSAFE,
      CreateMode.EPHEMERAL);
    System.out.println("Created " + createdPath);
  }
  
  public static void main(String[] args) throws Exception {
    JoinGroup joinGroup = new JoinGroup();
    joinGroup.connect(args[0]);
    joinGroup.join(args[1], args[2]);
    
    // stay alive until process is killed or thread is interrupted
    Thread.sleep(Long.MAX_VALUE);
  }
}
```
* 用于注册组成员的程序
    * 每个组成员将作为一个程序运行，并且加入到组中
    * 当程序退出时，组成员应当从组中被删除
    * 为了实现此，在ZooKeeper的命名空间中使用短暂znode来代表一个组成员
```
//ConnectionWatcher A helper class that waits for the connection to ZooKeeper to be established
import java.io.IOException;
import java.util.concurrent.CountDownLatch;

import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.Watcher.Event.KeeperState;

public class ConnectionWatcher implements Watcher {
  
  private static final int SESSION_TIMEOUT = 5000;

  protected ZooKeeper zk;
  private CountDownLatch connectedSignal = new CountDownLatch(1);

  public void connect(String hosts) throws IOException, InterruptedException {
    zk = new ZooKeeper(hosts, SESSION_TIMEOUT, this);
    connectedSignal.await();
  }
  
  @Override
  public void process(WatchedEvent event) {
    if (event.getState() == KeeperState.SyncConnected) {
      connectedSignal.countDown();
    }
  }
  
  public void close() throws InterruptedException {
    zk.close();
  }
}
```
* 等待与ZooKeeper建立连接的辅助类

```
% java JoinGroup localhost zoo duck
```
### 列出组成员
```
//ListGroup A program to list the members in a group
import java.util.List;

import org.apache.zookeeper.KeeperException;

public class ListGroup extends ConnectionWatcher {
    
  public void list(String groupName) throws KeeperException,
      InterruptedException {
    String path = "/" + groupName;
    
    try {
      List<String> children = zk.getChildren(path, false);
      if (children.isEmpty()) {
        System.out.printf("No members in group %s\n", groupName);
        System.exit(1);
      }
      for (String child : children) {
        System.out.println(child);
      }
    } catch (KeeperException.NoNodeException e) {
      System.out.printf("Group %s does not exist\n", groupName);
      System.exit(1);
    }
  }
  
  public static void main(String[] args) throws Exception {
    ListGroup listGroup = new ListGroup();
    listGroup.connect(args[0]);
    listGroup.list(args[1]);
    listGroup.close();
  }
}
```
* 用于列出组成员的程序
* 在list()方法中，调用getChildren()方法来检索并打印输出一个znode的子节点列表
    * 调用参数为znode的路径和观察标志
    * 设置了观察标志，znode的状态改变，关联的观察会被触发。
        * 查看znode节点时，通过设置观察可以让应用程序接收到组成员加入、退出和组被删除的有关通知
* 此程序中，捕捉了KeeperException.NoNodeException异常，代表组的znode不存在

```
% java ListGroup zoo
```
#### Zookeeper的命令行工具
Zookeeper有一套命令行工具。我们可以像如下使用，来查找zoo下的成员节点：
```
% zkCli.sh -server localhost ls /zoo
```
### 删除组
```
//DeleteGroup A program to delete a group and its members
import java.util.List;

import org.apache.zookeeper.KeeperException;

public class DeleteGroup extends ConnectionWatcher {
    
  public void delete(String groupName) throws KeeperException,
      InterruptedException {
    String path = "/" + groupName;
    
    try {
      List<String> children = zk.getChildren(path, false);
      for (String child : children) {
        zk.delete(path + "/" + child, -1);
      }
      zk.delete(path, -1);
    } catch (KeeperException.NoNodeException e) {
      System.out.printf("Group %s does not exist\n", groupName);
      System.exit(1);
    }
  }
  
  public static void main(String[] args) throws Exception {
    DeleteGroup deleteGroup = new DeleteGroup();
    deleteGroup.connect(args[0]);
    deleteGroup.delete(args[1]);
    deleteGroup.close();
  }
}
```
* 用于删除一个组及其所有成员的程序
* ZooKeeper提供了一个delete()方法，有两个参数：
    * 节点路径
    * 版本号
        * 如果所提供的版本号与znode的版本号一致，ZooKeeper会删除此znode
        * 通过将版本号设置为-1，可以绕过版本监测机制
    * 不支持递归的删除操作，删除父节点之前必须先删除子节点

```
% java DeleteGroup localhost zoo
```
## ZooKeeper 服务
* ZooKeeper是一个具有高可用性的高性能协调服务
* 从三个方面了解此服务：模型、操作和实现
### 数据模型
* ZooKeeper维护着一个树形层次结构，树的节点为znode
* ZooKeeper的数据访问具有原子性
    * 客户端在读取一个znode的数据时，要么读到所有的数据，要么读操作失败
    * 一个写操作将替换znode存储的所有数据，不成功就失败
    * 不支持添加操作
* znode用于存储数据，有一个与之相关联的ACL
* ZooKeeper被设计用来实现协调服务，而不是用于大容量数据存储，一个znode能存储的数据被限制在1MB以内
* znode通过路径被引用
    * 像Unix，在ZooKeeper中路径被表示成用斜杠分隔的Unicode字符串
    * ZooKeeper路径必须为规范的绝对路径
    * zookeeper为保留词，ZooKeeper使用/zookeeper子树来保存管理信息，如关于配额的信息
    * 通过Hadoop Path类来使用
* znode有一些性质非常适用于构建分布式应用

#### 短暂znode
* znode有两种类型：短暂的和持久的。类型创建时确定不能修改
* 短暂znode
    * 创建短暂znode的客户端会话结束时，ZooKeeper会删除之
    * 不可以有子节点
    * 虽然绑定到一个客户端会话，对所有客户端可见(需要符合其ACL定义)
    * 对于需要知道特定时刻有哪些分布式资源可用的应用，使用短暂znode是理想选择
* 持久znode
    * 不依赖于客户端会话
#### 顺序号
* 顺序(sequential)znode是指名称中包含ZooKeeper指定顺序号的znode
    * 如果在创建znode时设置了顺序标识，该znode名称之后便会附加一个值，此值由一个单调递增的计数器(父节点维护)添加
    * 如客户端请求创建一个名为/a/b-的顺序znode，所创建znode的名字可能是/a/b-3，稍后另一个顺序znode被创建，计数器会给出一个更大的值保证znode名称的唯一性，如/a/b-5
    * 在Java的API中，顺序znode的实际路径会作为create()调用的返回值传回客户端
* 顺序号可以用于为所有时间进行全局排序，这样客户端可以通过顺序号来推断事件的顺序
* 顺序号可以用于实现共享锁
#### 观察
* znode以某种方式发生改变时，“观察”(watch)机制可以让客户端得到通知
* 可以针对ZooKeeper服务的操作来设置观察，该服务的其他操作可以触发观察
    * 如客户端可以对一个znode调用exists操作，同时设定一个观察
        * 如果此znode不存在，客户端调用的exists操作将返回false
        * 一段时间后，另一个客户端创建了此znode，这次观察会被触发，通知前一个客户端
* 观察只能够触发一次(对连接事件的回调除外，这种观察不需要重新注册)
    * 为了能够多次收到通知，客户端需要重新注册所需的观察

### 操作
* ZooKeeper中有9种基本操作

| 操作 | 描述 |
| ------ | ------ |
| create | 创建一个znode(必须要有父节点) |
| delete | 删除一个znode(该znode不能有任何子节点) |
| exists | 测试一个znode是否存在并且查询它的元数据 |
| getACL,setACL | 获取/设置一个znode的ACL |
| getChildren | 获取一个znode的子节点列表 |
| getData,setData | 获取/设置一个znode所保存的数据 |
| sync | 将客户端的znode视图与ZooKeeper同步 |
* ZooKeeper的更新操作是有条件的
    * 在是有delete或setData操作时必须提供被更新znode的版本号(可通过exists操作获得)
    * 更新操作是非阻塞操作
        * 更新失败的客户端可以决定是否重试，或执行其他操作，并不会因此阻塞其他进程的执行
* ZooKeeper虽然可以看作是一个简单的文件系统，出于简单性的要求，有一些系统的基本操作被摒弃
    * ZooKeeper文件较小总是被整体读/写，没必要提供打开、关闭或查找操作
#### 集合更新
* ZooKeeper中有一个被称为multi的操作(Multiupdate)，用于将多个基本操作集合成一个操作单元
    * ZooKeeper确保这些基本操作同时被成功执行，或者同时失败
* 集合更新可以被用于在ZooKeeper中构建需要保持全局一致性的数据结构
#### 关于 API
* 对于ZooKeeper客户端，主要有两种语言绑定(binding)可以使用：Java和C
    * 也可以使用Perl、Python和REST的contrib绑定
* 对于每一种绑定语言来说，在执行操作时都可以选择同步执行或异步执行
* Java API同步执行的exists操作的签名
    ```
    public Stat exists(String path, Watcher watcher) throws KeeperException, InterruptedException
    ```
* Java API异步执行的exists操作的签名
    ```
    public void exists(String path, Watcher watcher, StatCallback cb, Object ctx)
    ```
* 所有异步操作的结果都是通过回调来传送的，因此Java API中异步方法的返回类型是void
* 异步操作调用者传递一个回调的实现，当ZooKeeper响应时，该回调方法被调用。在这种情况下，回调采用StatCallback接口，它有以下方法：
    ```
    public void processResult(int rc, String path, Object ctx, Stat stat);
    ```
    * rc为返回代码，对于KeeperException的代码。每个非零代码都代表一个异常，此情况下，stat参数是null
    * path和ctx参数对于客户端传递给exists()方法的参数，用于识别这个回调所响应的请求
#### 观察触发器
* 观察及其触发操作所对应的时间类型

| 设置观察的操作 | 创建znode | 创建子节点 | 删除znode | 删除子节点 | setData |
| ------ | ------ | ------ | ------ | ------ | ------ |
| exists | NodeCreated |  | NodeDeleted |  | NodeDataChanged |
| getData |  |  | NodeDeleted |  | NodeDataChanged |
| getChildren |  | NodeChildrenChanged | NodeDeleted | NodeChildrenChanged |  |

* 从收到观察事件到执行读操作期间，znode的状态可能会发生改变
#### ACL 列表
* 每个znode创建时都会带有一个ACL列表，用于决定谁可以对它执行何种操作
* ACL依赖于ZooKeeper的客户端身份验证机制。ZooKeeper提供以下几种身份验证机制：
    * digest——通过用户名和密码来识别客户端
    * sasl——通过Kerberos来识别客户端
    * ip——通过客户端的IP地址来识别客户端
* 在建立一个ZooKeeper会话后，客户端可以对自己进行身份验证
* 虽然znode的ACL列表会要求所有客户端是经过验证的，ZooKeeper的身份验证过程是可选的，客户端必须自己进行身份验证来支持对znode的访问
    * digest
        ```
        zk.addAuthInfo("digest", "tom:secret".getBytes());
        ```
* 每个ACL都是身份验证方式、符合该方式的一个身份和一组权限的组合
    * 给IP地址10.0.0.1的客户端对某个znode的读权限
        ```
        new ACL(Perm.READ, new ID("ip", "10.0.0.1"));
        ```
* ACL权限。 exists操作不受ACL权限的限制

| ACL权限 | 运行的操作 |
| ------ | ------ |
| create | create(子节点) |
| READ | getChildren、getData |
| WRITE | setData |
| DELETE | delete(子节点) |
| ADMIN | setACL |

* 在类ZooDefs.Ids中有一些预定义的ACL
* ZooKeeper还支持插入式身份验证机制，如需，可以集成第三方的身份验证系统
### 实现
* ZooKeeper有两种不同的运行模式：独立模式(standalone mode)和复制模式(replicated mode)
* 独立模式
    * 只有一个ZooKeeper服务器，适合于测试环境，不保证高可用性和可恢复性
* 复制模式
    * 生产环境中的ZooKeeper通常以复制模式运行于一个计算机集群上，此计算机集群被称为一个集合体(ensemble)
    * ZooKeeper通过复制来实现高可用性，只要集合体中半数以上的机器处于可用状态，就能够提供服务。出于此一个集合体通常包含奇数台机器
    * ZooKeeper确保对znode树的每一个修改都会被复制到集合体中超过半数的机器上
        * 此简单想法实现并不简单。ZooKeeper使用了Zab协议(Zookeeper Atomic Broadcast)，包括两个可以无限重复的阶段
#### 阶段1：领导者选举
* 集合体的所有机器通过一个选择过程来选出一台被称为领导者(leader)的机器，其他的机器被称为跟随者(follower)
    * 半数以上(或指定数量)的跟随者已经将其状态与领导者同步，则表明这个阶段已经完成
#### 阶段2：原子广播
* 所有的写请求都会被转发给领导者，再由领导者将更新广播给跟随者。当半数以上的跟随者已经将修改持久化之后，领导者才会提交这个更新，然后客户端才会收到一个更新成功的响应
    * 这个用来达成共识的协议被设计成具有原子性，因此每个修改要么成功要么失败
* 如果领导者出现故障，其余机器会选出另一个领导者，并和新的领导者一起继续提供服务
    * 如果之前的领导者恢复正常，会成为一个跟随者
    * 领导者选举的过程非常快，根据一个已公布的结果(http://bit.ly/dist_coordination)看，只需要大约200ms
* 在更新内存中的znode树之前，集合体中的所有机器都会先将更新写入磁盘
* 任一机器都可以为读请求提供服务，读请求只涉及内存检索，非常快
* 可以对ZooKeeper进行配置，使领导者不接受任何客户端连接。如此领导者的唯一任务是协调更新
    * leaderServes属性设置为no
    * 推荐超过3台服务器的集群中使用该设置
### 一致性
* 每一个对znode树的更新都被赋予一个全局唯一的ID，称为zxid(ZooKeeper Transaction ID)
* ZooKeeper要求对所有的更新进行编号并排序，它决定了分布式系统的执行顺序
* 在ZooKeeper的设计中，以下几点考虑保证了数据的一致性
    * 顺序一致性
        * 来自任意特定客户端的更新都会被按其发送顺序被提交
    * 原子性
        * 每个更新要么成功，要么失败
    * 单一系统映像
        * 一个客户端无论连接到哪一台服务器，看到的都是同样的系统视图
        * 当一台服务器出现故障，导致它的一个客户端需要尝试连接集合体中其他的服务器时，所有状态滞后于故障服务器的服务器都不会接受该连接请求，除非这些服务器将状态更新至故障服务器的水平
    * 持久性
        * 一个更新一旦成功，其结果就会持久存在并且不会被撤销
    * 及时性
        * 任何客户端所看到的滞后系统视图都是有限的，不会超过几十秒，意味着与其运行一个客户端看到非常陈旧的数据，还不如将服务器关闭，强迫该客户端连接到一个状态较新的服务器
        * 由于性能的原因，所有的读操作都是从ZooKeeper服务器的内存获得数据，它们不参与写操作的全局排序
        * 如果客户端之间通过ZooKeeper之间的机制进行通信，则客户端可能会发现它们所看到ZooKeeper状态是不一致的
            * 如客户端A将znode z的值从a更新为a'，接着A告诉B去读z的值，B读到的值是a而不是a'
            * 这与ZooKeeper的一致性保证是完全兼容的(这种情况称为“跨客户端视图的同时一致性”)
            * 为了避免这种情况，B应该在读之前对z调用sync操作
                * sync操作会强制B所连接的ZooKeeper服务器“赶上”领导者
                * sync操作只能以异步的方式调用。ZooKeeper会保证任何后续的操作都在服务器的sync操作完成后才执行
### 会话
* 每个ZooKeeper客户端的配置中都包括集合体中服务器的列表
* 在启动时，客户端会尝试连接到列表中的一台服务器，直到成功与一台服务器建立连接或因所有ZooKeeper服务器都不可用而失败
* 一旦客户端与一台ZooKeeper服务器建立连接，这台服务器就会为客户端创建一个新的会话
    * 每个会话都会有一个超时的时间设置，这个设置由创建会话的应用来设定
    * 如果服务器再超时时间段内没有收到任何请求，则相应会话会过期
    * 会话一旦过期，就无法重新打开，并且相关联的短暂znode都会丢失
    * 只要一个会话空闲超过一定时间，都可以通过客户端发送ping请求(也称为心跳)来保持会话不过期
        * ping请求是由ZooKeeper的客户端库自动发送，代码中不需要考虑如何维护会话
        * 此事件长度的设置应当足够低，以便能够检测出服务器故障(由读超时体现)，并能在会话超时的时间段内重新连接到另一台服务器
* ZooKeeper客户端可以自动地进行故障切换，切换至另一台ZooKeeper服务器
    * 在另一台服务器接替故障服务器之后，所有的会话(和相关的短暂znode)仍然是有效的
* 在故障切换过程中，应用程序将受到断开连接和连接至服务器的通知
    * 当客户端断开连接时，观察通知将无法发送；但是当客户端成功恢复连接后，这些通知还会被发送
    * 在客户端重新连接至另一台服务器的过程中，如果应用程序试图执行一个操作，该操作会失败
        * 在真实的ZooKeeper应用中处理连接丢失异常的重要性
#### 时间
* 在ZooKeeper中有几个时间参数
* “滴答”(tick time)参数定义了ZooKeeper中的基本时间周期，并被集合体中的服务器用来定义相互交互的时间表，其他设置都是根据滴答参数来定义的，或至少受它限制
    * 会话超时(session timeout)参数的值不可以小于2个滴答并且不大于20个滴答
    * 通常将滴答参数设置为2s(2000ms)
* 选择会话超时设置时有几点需要考虑
    * 较短的会话超时设置会较快地检测出机器故障
    * 要避免会话超时时间设得太低，繁忙的网络会导致数据包传输延迟，从而可能无意中导致会话超时
        * 此情况下，机器可能出现“振动”(flap)现象：在很短的时间内反复出现离开又重新加入组的情况
    * 创建较复杂暂时状态的应用程序，由于重建的代价较大，因此适合设置较长的会话超时
        * 某些情况下，可以对应用程序进行设计，使它能够在会话超时之前重启(适用于对应用进行维护或升级)
        * 服务器为每个会话分配一个唯一的ID和密码，如果在建立连接的过程中将它们传递给ZooKeeper，可以用于恢复一个会话(未超时)
        * 将会话ID和密码保存在稳定存储器中之后，可以将一个应用程序正常关闭，然后在重启应用之前凭借所保存的会话ID和密码来恢复会话环境
    * 一般的规则是，ZooKeeper集合体中的服务器越多，会话超时的设置应越大
    * 连接超时、读超时和ping周期都被定义为集合体中服务器数量的函数，因此集合体中服务器数量越多，这些参数值反而越小
    * 如果频繁遇到连接丢失的情况，应考虑增大超时的设置
* 可以使用JMX来监控ZooKeeper的度量指标，例如请求延迟的统计信息

### 状态
* ZooKeeper对象生命周期会经历几种不同的状态：CONNECTING、CONNECTED、COLOSED
* 可以在任何时刻通过getState()方法来查询对象的状态
    ```
    public States getState()
    ```
    * States定义为代表ZooKeeper对象不同状态的枚举类型值
    * 一个ZooKeeper的实例在一个时刻只能处于一种状态
* 通过注册观察对象，使用了ZooKeeper对象的客户端就可以收到状态转换通知
* 一个新建的ZooKeeper实例处于CONNECTING状态
* 一旦建立连接，会进入CONNECTED状态
    * 进入CONNECTED状态，观察对象会收到一个WatchedEvent通知，其中KeeperState的值是SyncConnected
* ZooKeeper实例可以断开然后重新连接到ZooKeeper服务，此时状态在CONNECTED和CONNECTING之间转换
    * 如果它断开连接，观察会受到一个Disconnected事件
* 如果close()方法被调用或出现会话超时，ZooKeeper实例会转换到CLOSED状态
    * 会话超时观察事件的KeeperState值为Expired
    * 处于CLOSED状态，ZooKeeper对象不再被认为是活跃的(可对States使用isAlive()测试)
    * 为重新连接到ZooKeeper服务，客户端必须创建一个新的ZooKeeper实例

## 使用 ZooKeeper 来构建应用
一定程度上了解ZooKeeper，下面用ZooKeeper写一些有用的应用程序
### 配置服务
* 配置服务是分布式应用所需要的基本服务之一，它使集群中的机器可以共享配置信息中那些公共的部分
    * 简单的说，ZooKeeper可以作为一个具有高可用性的配置存储器，允许分布式应用的参与者检索和更新配置文件
    * 使用ZooKeeper中的观察机制，可以建立一个活跃的配置服务，使那些感兴趣的客户端能够获得配置信息修改的通知
* 下面写一个这样的配置服务，通过两个假设简化所需实现的服务
    * 一，唯一需要存储的配置数据是字符串，关键字是znode的路径
    * 二，在任何时候只有一个客户端会执行更新操作

```
//== ActiveKeyValueStore
//== ActiveKeyValueStore-Read
//== ActiveKeyValueStore-Write
import java.nio.charset.Charset;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooDefs.Ids;
import org.apache.zookeeper.data.Stat;

public class ActiveKeyValueStore extends ConnectionWatcher {

  private static final Charset CHARSET = Charset.forName("UTF-8");

  public void write(String path, String value) throws InterruptedException,
      KeeperException {
    Stat stat = zk.exists(path, false);
    if (stat == null) {
      zk.create(path, value.getBytes(CHARSET), Ids.OPEN_ACL_UNSAFE,
          CreateMode.PERSISTENT);
    } else {
      zk.setData(path, value.getBytes(CHARSET), -1);
    }
  }
  public String read(String path, Watcher watcher) throws InterruptedException,
      KeeperException {
    byte[] data = zk.getData(path, watcher, null/*stat*/);
    return new String(data, CHARSET);
  }
}
```
* ActiveKeyValueStore类
    * write()方法的任务是将一个关键字及其值写入ZooKeeper
    * read()方法读取/config配置属性的值
        * getData()有三个参数：路径，观察对象，Stat。调用者可以获得一个znode的数据和元数据
            * 路径
            * 一个观察对象
            * Stat对象，由getData()方法返回的值填充，将信息回传给调用者


```
//ConfigUpdater An application that updates a property in ZooKeeper at random times
import java.io.IOException;
import java.util.Random;
import java.util.concurrent.TimeUnit;
import org.apache.zookeeper.KeeperException;

public class ConfigUpdater {
  
  public static final String PATH = "/config";
  
  private ActiveKeyValueStore store;
  private Random random = new Random();
  
  public ConfigUpdater(String hosts) throws IOException, InterruptedException {
    store = new ActiveKeyValueStore();
    store.connect(hosts);
  }
  
  public void run() throws InterruptedException, KeeperException {
    while (true) {
      String value = random.nextInt(100) + "";
      store.write(PATH, value);
      System.out.printf("Set %s to %s\n", PATH, value);
      TimeUnit.SECONDS.sleep(random.nextInt(10));
    }
  }
  
  public static void main(String[] args) throws Exception {
    ConfigUpdater configUpdater = new ConfigUpdater(args[0]);
    configUpdater.run();
  }
}
```
* ConfigUpdater随机更新ZooKeeper中配置属性值，用于说明ActiveKeyValueStore的用法

```
//ConfigWatcher An application that watches for updates of a property in ZooKeeper and prints them to the console
import java.io.IOException;

import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.Watcher.Event.EventType;

public class ConfigWatcher implements Watcher {
  
  private ActiveKeyValueStore store;
  
  public ConfigWatcher(String hosts) throws IOException, InterruptedException {
    store = new ActiveKeyValueStore();
    store.connect(hosts);
  }
  
  public void displayConfig() throws InterruptedException, KeeperException {
    String value = store.read(ConfigUpdater.PATH, this);
    System.out.printf("Read %s as %s\n", ConfigUpdater.PATH, value);
  }

  @Override
  public void process(WatchedEvent event) {
    if (event.getType() == EventType.NodeDataChanged) {
      try {
        displayConfig();
      } catch (InterruptedException e) {
        System.err.println("Interrupted. Exiting.");        
        Thread.currentThread().interrupt();
      } catch (KeeperException e) {
        System.err.printf("KeeperException: %s. Exiting.\n", e);        
      }
    }
  }
  
  public static void main(String[] args) throws Exception {
    ConfigWatcher configWatcher = new ConfigWatcher(args[0]);
    configWatcher.displayConfig();
    
    // stay alive until process is killed or thread is interrupted
    Thread.sleep(Long.MAX_VALUE);
  }
}
```
* 作为配置服务的用户，ConfigWatcher创建了ActiveKeyValueStore对象store，并且在启动后调用store的read()方法，将自身作为观察传递给store
    * displayConfig()方法用于显示它说所读到的配置信息的初始值
    * 当ConfigUpdater更新znode时，ZooKeeper产生一个类型为EventType.NodeDataChanged的事件，从而触发观察。ConfigWatcher在它的process()方法中对该事件做出反应
    * 由于观察仅发送单次信号，因此每次调用ActiveKeyValueStore的read()方法时，都将一个新的观察告知ZooKeeper
        * 仍然不能保证接收到每一个更新，收到观察事件通知与下一个读之间，znode可能已经被更新

```
# 使用此程序
# 在一个终端窗口中运行configUpdate
% java configUpdate localhost
Set /config to 79
Set /config to 14
Set /config to 78
# 紧接着在另一个终端窗口启动configWatcher
% java configWatcher localhost
Read /config to 79
Read /config to 14
Read /config to 78
```

### 可复原的 ZooKeeper 应用
* 关于分布式计算(http://bit.ly/dist_computing)的第一个误区是“网络是可靠的”
* 在Java API中的每一个ZooKeeper操作都在其throws子句中声明了两种类型的异常：
    * InterruptedException
    * KeeperException

#### InterruptedException 异常
* 如果操作被中断，则会有一个InterruptedException异常被抛出
    * Java中有一个取消阻塞方法的标准机制，即针对阻塞方法的线程调用Interrupt()
        * 一个成功的取消操作将产生一个InterruptedException异常
         * ZooKeeper也遵循这一机制，可以用此方法取消一个ZooKeeper操作。使用了ZooKeeper的类或库通常会传播InterruptedException异常。使客户端能够取消它们的操作
* InterruptedException异常并不意味着有故障，而是表明响应的操作已经被取消，可以通过传播异常来中止应用程序的运行

#### KeeperException 异常
* ZooKEeper服务器发出一个错误信号或服务器存在通信问题，抛出的则是KeeperException异常
* 针对不同的错误情况，KeeperException异常存在不同的子类
    * KeeperException.NoNodeException是KeeperException的一个子类，针对不存在的znode操作抛出此异常
    * 每一个KeeperException异常的子类都对应一个关于错误类型信息的代码
        * KeeperException.NoNodeException的异常代码是KeeperException.Code.NONODE(一个枚举值)
* 两种方法被用来处理KeeperException异常
    * 捕捉KeeperException异常并且通过检测它的代码来决定采取何种补救措施
    * 捕捉等价的KeeperException子类并且在每段捕捉代码中执行响应的操作
* KeeperException异常分为三类：
    * 状态异常——当一个操作因不能被应用于znode树而被导致失败时，就会出现状态异常
        * 如果一个znode先被另一个进程更新了，根据版本号执行setData操作的进程就会失败，并收到KeeperException.BadVersionException异常
        * 一些状态异常会指出程序中的错误，如KeeperException.NoChildrenForEphemeralsException
    * 可恢复的异常——可恢复的异常是指那些应用程序能够在同一个ZooKeeper会话中恢复的异常
        * 通过KeeperException.ConnectionLossException来表示
            * 丢失了与ZooKeeper的连接。ZooKeeper会尝试重新连接，大多数情况下重新连接会成功，并确保会话完整
            * ZooKeeper不能判断与此异常相关的操作是否成功执行(部分失败)
                * 程序员有责任来解决这种不确定性，根据应用的情况采取适当的操作
                * 需要对幂等(idempotent)操作和非幂等(Nonidempotent)操作进行区分
                    * 幂等：一次或多次操作都会产生相同结果的操作
    * 不可恢复的异常——在某些情况下，ZooKeeper会话会失效，因超时或会话被关闭(KeeperException.SessionExpiredException)，或因身份验证失败(KeeperException.AuthFailException)。上述情况下，所有与会话相关联的短暂znode都将丢失，应用程序需要在重新连接到ZooKeeper之前重建它的状态

#### 可靠的配置服务
```
  private static final int MAX_RETRIES = 5;
  private static final int RETRY_PERIOD_SECONDS = 10;
  //ResilientActiveKeyValueStore-Write
  public void write(String path, String value) throws InterruptedException,
      KeeperException {
    int retries = 0;
    while (true) {
      try {
        Stat stat = zk.exists(path, false);
        if (stat == null) {
          zk.create(path, value.getBytes(CHARSET), Ids.OPEN_ACL_UNSAFE,
              CreateMode.PERSISTENT);
        } else {
          zk.setData(path, value.getBytes(CHARSET), stat.getVersion());
        }
        return;
      } catch (KeeperException.SessionExpiredException e) {
        throw e;
      } catch (KeeperException e) {
        if (retries++ == MAX_RETRIES) {
          throw e;
        }
        // sleep then retry
        TimeUnit.SECONDS.sleep(RETRY_PERIOD_SECONDS);
      }
    }
  }
```
* 对ActiveKeyValueStore的write()修改，使这个幂等方法能够循环执行重试
    * 设置了重试的最大次数MAX_RETRIES和两次重试之间的时间间隔RETRY_PERIOD_SECONDS
    * 另一种重试处理策略：“指数退回”(exponential backoff)，每次讲重试的间隔乘以一个常数

```
public class ResilientConfigUpdater {

  ...
  
  //ResilientConfigUpdater
  public static void main(String[] args) throws Exception {
    while (true) {
      try {
        ResilientConfigUpdater configUpdater =
          new ResilientConfigUpdater(args[0]);
        configUpdater.run();
      catch (KeeperException.SessionExpiredException e) {
        // start a new session
      } catch (KeeperException e) {
        // already retried, so exit
        e.printStackTrace();
        break;
      }
    }
  }
}
```
* 对KeeperException.SessionExpiredException进行处理的两种方式：
    * 上创建一个新的ZooKeeper实例
    * 在观察中(例子中是ConnectionWatcher)检测类型为Expired的KeeperState，然后在检测到的时候创新一个连接
* 忽略了另一种故障模式
    * ZooKeeper对象被创建，尝试连接一个ZooKeeper服务器。连接失败或超时，尝试连接集合体中的另一台服务器，尝试所有服务器仍无法建立连接，抛出IOException异常

### 锁服务
* 分布式锁能够在一组进程之间提供互斥机制，使得在任何时刻都只有一个进程可以持有锁
* 分布式锁可以用于在大型分布式系统中实现领导者选举，在任何时间点，持有锁的那个进程就是系统的领导者
    * 不要将ZooKeeper自己的领导者选举(不对外公开)和使用ZooKeeper基本操作的一般的领导者选举服务混为一谈
* 下面用ZooKeeper来实现分布式锁服务
    * 使用顺序znode来为那些竞争锁的进程强制排序
    * 首先制定一个作为锁的znode，通常用它来描述被锁定的实体，称为/leader
    * 然后希望获得锁的客户端创建一些短暂顺序znode，作为锁znode的子节点
    * 在任何时间点，顺序号最小的客户端将持有锁
        * 如两个客户端差不多同时创建znode，分别为/leader/lock-1和/leader/lock-2，创建1的客户端会持有锁
    * 通过删除znode 如/leader/lock-1即可简单地将锁释放
    * 通过创建一个关于znode删除的观察，可以使客户端在获得锁时得到通知
* 申请锁的伪代码：
    * 在锁znode下创建一个名为lock-的短暂顺序znode，并且记住它的实际路径名
    * 查询锁znode的子节点并且设置一个观察
    * 如果步骤一中所创建的znode在步骤二返回的所有子节点中具有最小的顺序号，则获得锁，退出
    * 等待步骤二中所设观察的通知，转到步骤二

#### 羊群效应
* 上述算法正确，还存在一些问题
* 第一个问题是这种实现会受到羊群效应(herd effect)的影响
    * ”羊群效应“是指大量客户端收到同一事件的通知，但实际上只有很少一部分需要处理这一事件
* 在成百上千的客户端的情况下，所有客户端都在尝试获得锁，每个客户端都会在锁znode上设置一个观察，用于捕捉子节点的变化。每次锁被释放或一个新进程开始申请锁，观察都会被触发并且每个客户端都会收到一个通知。只有一个客户端会成功地获取锁，但维护的过程以及向所有客户端发送观察事件会产生峰值流量，对ZooKeeper服务器造成压力
* 需要优化发送通知的条件
    * 关键在于仅当前一个顺序号的子节点消失时才通知下一个客户端

#### 可恢复的异常
* 另一个问题：不能处理因连接丢失而导致的create操作失败
* 在重新连接之后客户端不能判断它是否已经创建过子节点
* 解决方案是在znode的名称中嵌入一个ID
    * 重新连接后，客户端可以对锁节点的所有子节点进行检查，是否有子节点的名称包含ID
    * 客户端会话的ID是一个长整数，在ZooKeeper服务中唯一，适合在连接丢失后用于重新识别客户端
        * 调用Java ZooKeeper类的getSessionId()方法获得会话的ID
    * 创建短暂顺序znode采用lock-\<sessionId\>-\<sequenceNumber\>
#### 不可恢复的异常
* 会话过期，创建的短暂znode将会被删除，已持有的锁被释放，或者是放弃了申请锁的位置
* 使用锁的应用程序应当意识到已经不再持有锁，应当清理状态，然后通过创建并尝试一个新的锁对象来重新启动
#### 实现
* ZooKeeper带有一个Java语言写的生产级别的锁实现，名为writelock，客户端可以很方便地使用
### 更多分布式数据结构和协议
* 使用ZooKeeper可以实现很多不同的分布式数据结构和协议，例如：
    * “屏障”(barrier)
    * 队列
    * 两段提交协议
    * 上述均为同步协议，但可以使用异步ZooKeeper基本操作来实现
* ZooKeeper网站(http://zookeeper.apache.org)提供了一些用于实现分布式数据结构和协议的伪代码
* ZooKeeper本身带有一些标准方法的实现(包括锁、领导者选举和队列)，放在安装位置下的recipes目录中
* Curator项目(https://github.com/Netflix/curator)提供了更多ZooKeeper方法的实现
#### BookKeeper 和 Hedwing
* BookKeeper是一个具有高可用性和可靠性的日志服务
* BookKeeper可以用来实现预写式日志(write-ahead logging)
    * 预写式日志是一项在存储系统中用于保证数据完整性的常用技术
    * 在一个使用预写式日志的系统中，每一个写操作在被应用前都先要写入事务日志
    * 使用此技术，不必在每个写操作之后都将数据写到永久存储器上
* BookKeeper客户端所创建的日志被称为ledger，每一个添加到ledger的记录被称为ledger entry
    * 每个ledger entry就是一个简单的字节数组
    * ledger由保存有ledger数据副本的bookie服务器组进行管理
    * ledger数据不存储在ZooKeeper中，只有元数据保存在ZooKeeper中
* Hedwing是利用BookKeeper实现的一个基于主题的发布-订阅系统
* BookKeeper是ZooKeeper的一个子项目，Http://zookeeper.apache.org/bookkeeper/
## 生产环境中的 ZooKeeper
* 在生产环境中，ZooKeeper应当以复制模式运行
* 下面讨论使用ZooKeeper服务器的集合体时需要考虑的一些问题
* 参考《ZooKeeper 管理员指南》(http://bit.ly/admin_guide)获得详细的最新操作指南包括
    * 支持的平台
    * 推荐的硬件
    * 维护过程
    * 配置属性
### 可恢复性和性能
* 安放ZooKeeper所用的机器时，应当考虑尽量减少机器和网络故障可能带来的影响
    * 实践过程中，一般是跨机架、电源和交换机来安放服务器，任何一个出现故障不会使集合体损失半数以上的服务器
* 对于需要低延迟服务(毫秒级别)的应用，最好所有服务器在同一个数据中心的同一个集合体中
* 对于不需要低延迟服务的应用，可以通过跨数据中心安防服务器来获得更好的可恢复性
    * 领导者选举和分布式粗粒度锁是这类应用的代表
* 观察节点指没有投票权的跟随节点
    * 观察节点不参与写请求过程中达成共识的投票，因此使用观察节点可以让ZooKeeper集群在不影响写性能的情况下提高操作的性能
    * 使用观察节点可以让ZooKeeper集群跨越多个数据中心，而不增加正常投票节点的延迟
        * 可以将投票节点安放在一个数据中心，将观察节点安放在另一个数据中心来实现
* ZooKeeper是具有高可用性的系统。应当运行在专用机器上
* 通过对ZooKeeper进行配置，可以使它的事务日志和数据快照分别保存在不同的磁盘驱动器上
    * 默认情况下，两者都保存在dataDir属性所指定的目录中
    * 通过为dataLogDir设置一个值，便可以将事务日志写在指定的位置
    * 通过指定一个专用的设备(不只是一个分区)，一个ZooKeeper服务器可以以最大速率将日志记录写到磁盘
        * 写日志是顺序写，没有寻址操作
* 由于所有的写操作都是通过领导者来完成的，增加服务器并不能提高写操作的吞吐量，提高写操作的关键是写操作的速度
* 如果写操作的进程被交换到磁盘上，则性能会受到不利影响，通过将Java堆的大小设置为小于机器上空闲的物理内存即可
    * ZooKeeper脚本可以从它的配置目录中获取一个名为java.env的文件，这个文件被用来设置JVMFLAGS环境变量，包括设置Java堆的大小(和任何其他所需的JVM参数)
### 配置
* ZooKeeper服务器的集合体中，每个服务器都有一个集合体中唯一的数值型的ID，取值范围在1~255之间
    * 可以通过一个名为myId的纯文本文件设定服务器的ID，文件保存在dataDir参数指定的目录中
* 为每台服务器设置ID只完成了工作的一般。还需要将集合体中其他服务器的ID和网络位置告诉其他服务器。在ZooKeeper的配置文件中必须为每台服务器添加下面这行配置：
    ```
    server.n=hostname:port:port
    ```
    * n是服务器的ID
    * 第一个port是跟随者用来连接领导者的端口
    * 第二个端口用于领导者选举
* 下面是一个包含三台机器的复制模式下ZooKeeper集合体的配置例子：
    ```
    tickTime=2000
    dataDir=/disk1/zookeeper
    dataLogDir=/disk2/zookeeper
    clientPort=2181
    initLimit=5
    syncLimit=2
    server.1=zookeeper1:2888:3888
    server.2=zookeeper2:2888:3888
    server.3=zookeeper3:2888:3888
    ```
    * 连接到此ZooKeeper集合体的客户端在ZooKeeper对象的构造函数中应当使用zookeeper1:2181、zookeeper2:2181和zookeeper3:2181作为主机字符串
    * 复制模式下，有两个额外的参数：initLimit和syncLimit，两者都是以滴答参数的倍数进行度量
        * initLimit设定了所有跟随者与领导者进行连接并同步的时间范围
            * 在设定的时间段内半数以上的跟随者未能完成同步，领导者便会宣布放弃领导者地位，然后进行另一次领导者选举
        * syncLimit设定了运行一个跟随者与领导者进行同步的时间
            * 在设定的时间段内一个跟随者未能完成同步，会自己重启。所有关联到跟随者的客户端将连接到另一个跟随者
* 以上是建立和运行一个ZooKeeper服务器集群所需的最少参数，《ZooKeeper管理员指南》列出了更多的配置选项，特别是性能调优方面的

## 延伸
* 《ZooKeeper》：http://shop.oreilly.com/product/0636920028901.do
