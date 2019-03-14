<!-- TOC -->

- [Hadoop 运行环境搭建——Ubuntu](#hadoop-运行环境搭建ubuntu)
    - [Java 环境安装](#java-环境安装)
        - [使用ppa/源方式安装](#使用ppa源方式安装)
        - [直接下载jdk压缩包方式安装(这里只介绍jdk7的,jdk8的原理完全一致)](#直接下载jdk压缩包方式安装这里只介绍jdk7的jdk8的原理完全一致)
    - [ssh 安装](#ssh-安装)
    - [hadoop 安装](#hadoop-安装)
        - [创建hadoop用户](#创建hadoop用户)
        - [安装 Hadoop 2](#安装-hadoop-2)
        - [Hadoop单机配置(非分布式)](#hadoop单机配置非分布式)
        - [Hadoop伪分布式配置](#hadoop伪分布式配置)
        - [YARN 运行](#yarn-运行)
        - [Hadoop集群安装配置](#hadoop集群安装配置)
            - [网络配置](#网络配置)
            - [配置集群/分布式环境](#配置集群分布式环境)

<!-- /TOC -->
## Hadoop 运行环境搭建——Ubuntu

### Java 环境安装
1) 通过ppa(源) 方式安装.可以通过 apt-get upgrade 方式方便获得jdk的升级.
2) 通过官网下载安装包安装.
#### 使用ppa/源方式安装
1) 添加ppa
    ```
    sudo add-apt-repository ppa:webupd8team/java
    sudo apt-get update
    ```
2) 安装oracle-java-installer
    ```
    sudo apt-get install oracle-java8-installer #jdk8 安装路径:/usr/lib/jvm/java-8-oracle
    ```
    安装器会提示是否同意 oracle 的服务条款,选择 ok, 可以加入下面的这条命令,默认同意条款.
    ```
    echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | sudo /usr/bin/debconf-set-selections  #JDK8 默认选择条款
    ```
    如果因为防火墙或者其他原因,导致installer下载速度很慢,可以中断操作.然后下载好相应jdk的tar.gz 包, /var/cache/oracle-jdk8-installer (jdk8) 下面,然后安装一次installer.installer则会默认使用下载的tar.gz包
3) 设置系统默认jdk
    JDK8:sudo update-java-alternatives -s java-8-oracle
    如果既安装了jdk7,又安装了jdk8,要实现两者的切换,可
    jdk8切换到jdk7:sudo update-java-alternatives -s java-7-oracle
    jdk7切换到jdk8:sudo update-java-alternatives -s java-8-oracle
4) 测试jdk 是是否安装成功:
    ```
    java -version
    javac -version
    ```
#### 直接下载jdk压缩包方式安装(这里只介绍jdk7的,jdk8的原理完全一致)
1) 官网下载JDK,选择相应的.gz包下载 
    地址: http://www.oracle.com/technetwork/articles/javase/index-jsp-138363.html
2) 解压缩,放到指定目录(以jdk-7u60-linux-x64.gz为例)
    ```
    sudo mkdir /usr/lib/jvm #创建目录
    sudo tar -zxvf jdk-7u60-linux-x64.gz -C /usr/lib/jvm #解压缩到该目录
    ```
3) 修改环境变量
    ```
    sudo vim ~/.bashrc (/etc/profile)
    ```
    文件的末尾追加下面内容:
    ```
    #set oracle jdk environment
    export JAVA_HOME=/usr/lib/jvm/jdk1.7.0_60
    export JRE_HOME=${JAVA_HOME}/jre
    export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
    export PATH=${JAVA_HOME}/bin:$PATH
    ```
    使环境变量马上生效:
    ```
    source ~/.bashrc (/etc/profile)
    ```
5) 设置系统默认jdk 版本
    ```
    sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk1.7.0_60/bin/java 300  
    sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/jdk1.7.0_60/bin/javac 300  
    sudo update-alternatives --install /usr/bin/jar jar /usr/lib/jvm/jdk1.7.0_60/bin/jar 300   
    sudo update-alternatives --install /usr/bin/javah javah /usr/lib/jvm/jdk1.7.0_60/bin/javah 300   
    sudo update-alternatives --install /usr/bin/javap javap /usr/lib/jvm/jdk1.7.0_60/bin/javap 300   
    ```
    然后执行:
    ```
    sudo update-alternatives --config java
    ```
    若是初次安装jdk,会有下面的提示     
    There is only one alternative in link group java (providing /usr/bin/java): /usr/lib/jvm/jdk1.7.0_60/bin/java
否则,选择合适的jdk
5) 测试jdk
    ```
    java -version
    ```
### ssh 安装
```
sudo apt-get install openssh-server
```
### hadoop 安装
#### 创建hadoop用户
如果安装 Ubuntu 的时候不是用的 “hadoop” 用户，那么需要增加一个名为 hadoop 的用户。输入如下命令创建新用户:
```
sudo useradd -m hadoop -s /bin/bash(创建可以登陆的 hadoop 用户，并使用 /bin/bash 作为 shell)
sudo passwd hadoop(设置密码)
sudo adduser hadoop sudo(添加权限)
```
#### 安装 Hadoop 2
1) 将 Hadoop 安装至 /usr/local/ 中:
    ```
    sudo tar -zxf ~/下载/hadoop-2.6.0.tar.gz -C /usr/local    # 解压到/usr/local中
    cd /usr/local/
    sudo mv ./hadoop-2.6.0/ ./hadoop  # 将文件夹名改为hadoop
    sudo chown -R hadoop ./hadoop     # 修改文件权限
    cd /usr/local/hadoop
    ./bin/hadoop version              #检查 Hadoop 是否可用，成功则会显示 Hadoop 版本信息
    ```
2) 环境变量设置
    ```
    vi /etc/profile
    ```
    ```
    #末尾添加HADOOP_HOME
    export HADOOP_HOME=/usr/local/hadoop
    export PATH=$PATH:$HADOOP_HOME/bin
    export PATH=$PATH:$HADOOP_HOME/sbin
    ```
#### Hadoop单机配置(非分布式)
Hadoop 默认模式为非分布式模式（本地模式），无需进行其他配置即可运行。非分布式即单 Java 进程，方便进行调试。
#### Hadoop伪分布式配置
Hadoop 可以在单节点上以伪分布式的方式运行，Hadoop 进程以分离的 Java 进程来运行，节点既作为 NameNode 也作为 DataNode，同时，读取的是 HDFS 中的文件。
Hadoop 的配置文件位于 /usr/local/hadoop/etc/hadoop/ 中，伪分布式需要修改2个配置文件 core-site.xml 和 hdfs-site.xml 。Hadoop的配置文件是 xml 格式，每个配置以声明 property 的 name 和 value 的方式来实现。
Hadoop 的运行方式是由配置文件决定的（运行 Hadoop 时会读取配置文件），因此如果需要从伪分布式模式切换回非分布式模式，需要删除 core-site.xml 中的配置项。
此外，伪分布式虽然只需要配置 fs.defaultFS 和 dfs.replication 就可以运行（官方教程如此），不过若没有配置 hadoop.tmp.dir 参数，则默认使用的临时目录为 /tmp/hadoo-hadoop，而这个目录在重启时有可能被系统清理掉，导致必须重新执行 format 。所以进行了设置，同时也指定 dfs.namenode.name.dir 和 dfs.datanode.data.dir，否则可能会出错。
1) 修改配置文件 core-site.xml 
    ```
    <configuration>
        <property>
            <name>hadoop.tmp.dir</name>
            <value>file:/usr/local/hadoop/tmp</value>
            <description>Abase for other temporary directories.</description>
        </property>
        <property>
            <name>fs.defaultFS</name>
            <value>hdfs://localhost:9000</value>
        </property>
    </configuration>
    ```
2) 修改配置文件 hdfs-site.xml
    ```
    <configuration>
        <property>
            <name>dfs.replication</name>
            <value>1</value>
        </property>
        <property>
            <name>dfs.namenode.name.dir</name>
            <value>file:/usr/local/hadoop/tmp/dfs/name</value>
        </property>
        <property>
            <name>dfs.datanode.data.dir</name>
            <value>file:/usr/local/hadoop/tmp/dfs/data</value>
        </property>
    </configuration>
    ```
3) 执行 NameNode 的格式化
    ```
    ./bin/hdfs namenode -format
    ```
4) 开启 NameNode 和 DataNode 守护进程
    ```
    ./sbin/start-dfs.sh
    ```
5) 通过命令 jps 来判断是否成功启动
    ```
    jps
    ```
成功启动后，可以访问 Web 界面 http://localhost:50070 查看 NameNode 和 Datanode 信息，还可以在线查看 HDFS 中的文件。
#### YARN 运行
YARN 是从 MapReduce 中分离出来的，负责资源管理与任务调度。YARN 运行于 MapReduce 之上，提供了高可用性、高扩展性，YARN 的更多介绍在此不展开，有兴趣的可查阅相关资料。
上述通过 ./sbin/start-dfs.sh 启动 Hadoop，仅仅是启动了 MapReduce 环境，我们可以启动 YARN ，让 YARN 来负责资源管理与任务调度。
YARN 主要是为集群提供更好的资源管理与任务调度，然而这在单机上体现不出价值，反而会使程序跑得稍慢些。因此在单机上是否开启 YARN 就看实际情况了。
如果不想启动 YARN，务必把配置文件 mapred-site.xml 重命名，改成 mapred-site.xml.template，需要用时改回来就行。否则在该配置文件存在，而未开启 YARN 的情况下，运行程序会提示 “Retrying connect to server: 0.0.0.0/0.0.0.0:8032” 的错误，这也是为何该配置文件初始文件名为 mapred-site.xml.template。
1) 修改配置文件 mapred-site.xml
    ```
    mv ./etc/hadoop/mapred-site.xml.template ./etc/hadoop/mapred-site.xml
    vi ./etc/hadoop/mapred-site.xml
    ```
    ```
    <configuration>
        <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
        </property>
    </configuration>
    ```
2) 修改配置文件 yarn-site.xml
    ```
    <configuration>
        <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
            </property>
    </configuration>
    ```
3) 启动 YARN
    ```
    ./sbin/start-dfs.sh
    ./sbin/start-yarn.sh      # 启动YARN
    ./sbin/mr-jobhistory-daemon.sh start historyserver  # 开启历史服务器，才能在Web中查看任务运行情况
    ```
4) 关闭 YARN
    ```
    ./sbin/stop-yarn.sh
    ./sbin/mr-jobhistory-daemon.sh stop historyserver
    ```
#### Hadoop集群安装配置
使用 Ubuntu 14.04 64位 作为系统环境，基于原生 Hadoop 2，在 Hadoop 2.6.0 (stable) 版本下验证通过，可适合任何 Hadoop 2.x.y 版本。简单的使用两个节点作为集群环境: 一个作为 Master 节点，局域网 IP 为 192.168.1.121；另一个作为 Slave 节点，局域网 IP 为 192.168.1.122。
Hadoop 集群的安装配置大致为如下流程:
    1) 选定一台机器作为 Master
    2) 在 Master 节点上配置 hadoop 用户、安装 SSH server、安装 Java 环境
    3) 在 Master 节点上安装 Hadoop，并完成配置
    4) 在其他 Slave 节点上配置 hadoop 用户、安装 SSH server、安装 Java 环境
    5) 将 Master 节点上的 /usr/local/hadoop 目录复制到其他 Slave 节点上
    6) 在 Master 节点上开启 Hadoop
##### 网络配置
假设集群所用的节点都位于同一个局域网.
如果使用的是虚拟机安装的系统，那么需要更改网络连接方式为桥接（Bridge）模式，才能实现多个节点互连，要确保各个节点的 Mac 地址不同(否则 IP 会冲突).
1) 首先在 Master 节点上完成准备工作，并关闭 Hadoop (/usr/local/hadoop/sbin/stop-dfs.sh)，再进行后续集群配置。
2) 修改各个节点的主机名,修改节点的IP映射
    1) 为了便于区分，可以修改各个节点的主机名（在终端标题、命令行中可以看到主机名，以便区分）。
        在 Ubuntu/CentOS 7 中，我们在 Master 节点上执行如下命令修改主机名(即改为 Master，区分大小写):
        ```
        sudo vim /etc/hostname
        ```
        添加
        ```
        HOSTNAME=Master
        ```
    2) 执行如下命令修改所用节点的IP映射
        ```
        sudo vim /etc/hosts
        ```
        例如:
        ```
        192.168.1.121   Master
        193.192.168.1.122   Slave1
        ```
        修改完成后需要重启一下，重启后在终端中才会看到机器名的变化。
        在所有节点上完成网络配置
    3) 测试是否相互 ping 得通
        配置好后需要在各个节点上执行如下命令，测试是否相互 ping 得通，如果 ping 不通，后面就无法顺利配置成功：
        ```
        ping Master -c 3   # 只ping 3次，否则要按 Ctrl+c 中断
        ping Slave1 -c 3
        ```
3) SSH无密码登陆节点
    这个操作是要让 Master 节点可以无密码 SSH 登陆到各个 Slave 节点上。
    1) 生成 Master 节点的公匙
        在 Master 节点的终端中执行(因为改过主机名，所以还需要删掉原有的再重新生成一次):
        ```
        cd ~/.ssh               # 如果没有该目录，先执行一次ssh localhost
        rm ./id_rsa*            # 删除之前生成的公匙（如果有）
        ssh-keygen -t rsa       # 一直按回车就可以
        ```
    2) Master 节点无密码 SSH 本机
        在 Master 节点上执行:
        ```
        cat ./id_rsa.pub >> ./authorized_keys
        ```
        完成后可执行命令验证:
        ```
        ssh Master
        ```
    3) 公匙传输
        在 Master 节点将上公匙传输到 Slave1 节点:
        ```
        scp ~/.ssh/id_rsa.pub hadoop@Slave1:/home/hadoop/
        ```
    4) 公匙授权
    在 Slave1 节点上:
        ```
        mkdir ~/.ssh       # 如果不存在该文件夹需先创建，若已存在则忽略
        cat ~/id_rsa.pub >> ~/.ssh/authorized_keys
        rm ~/id_rsa.pub    # 用完就可以删掉了
        ```
        如果有其他 Slave 节点，也要执行将 Master 公匙传输到 Slave 节点、在 Slave 节点上加入授权这两步。这样，在 Master 节点上就可以无密码 SSH 到各个 Slave 节点了，可在 Master 节点上执行如下命令进行检验:
        ```
        ssh Slave1
        ```
##### 配置集群/分布式环境
1) 配置文件修改
    集群/分布式模式需要修改 /usr/local/hadoop/etc/hadoop 中的5个配置文件，更多设置项可查看官方说明，这里仅设置了正常启动所必须的设置项： slaves、core-site.xml、hdfs-site.xml、mapred-site.xml、yarn-site.xml 。
    1) 文件 slaves 配置
        文件 slaves，将作为 DataNode 的主机名写入该文件，每行一个，默认为 localhost，所以在伪分布式配置时，节点即作为 NameNode 也作为 DataNode。分布式配置可以保留 localhost，也可以删掉，让 Master 节点仅作为 NameNode 使用。本教程让 Master 节点仅作为 NameNode 使用，因此将文件中原来的 localhost 删除，只添加一行内容：Slave1。
    2) 文件 core-site.xml 配置
    ```
    <configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://Master:9000</value>
        </property>
        <property>
                <name>hadoop.tmp.dir</name>
                <value>file:/usr/local/hadoop/tmp</value>
                <description>Abase for other temporary directories.</description>
        </property>
    </configuration>
    ```
    3) 文件 hdfs-site.xml 配置，dfs.replication 一般设为3，但只一个 Slave 节点，所以 dfs.replication 的值设为1
    ```
    <configuration>
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>Master:50090</value>
        </property>
        <property>
                <name>dfs.replication</name>
                <value>1</value>
        </property>
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:/usr/local/hadoop/tmp/dfs/name</value>
        </property>
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:/usr/local/hadoop/tmp/dfs/data</value>
        </property>
    </configuration>
    ```
    4) 文件 mapred-site.xml (可能需要先重命名，默认文件名为 mapred-site.xml.template)配置
    ```
    <configuration>
            <property>
                    <name>mapreduce.framework.name</name>
                        <value>yarn</value>
            </property>
            <property>
                    <name>mapreduce.jobhistory.address</name>
                    <value>Master:10020</value>
            </property>
            <property>
                    <name>mapreduce.jobhistory.webapp.address</name>
                    <value>Master:19888</value>
            </property>
    </configuration>
    ```
    5) 文件 yarn-site.xml 配置
    ```
    <configuration>
            <property>
                    <name>yarn.resourcemanager.hostname</name>
                    <value>Master</value>
            </property>
            <property>
                    <name>yarn.nodemanager.aux-services</name>
                    <value>mapreduce_shuffle</value>
            </property>
    </configuration>
    ```
2) 初始化
    配置好后，将 Master 上的 /usr/local/Hadoop 文件夹复制到各个节点上。在切换到集群模式前先删除之前的临时文件。
    在 Master 节点上执行：
    ```
    cd /usr/local
    sudo rm -r ./hadoop/tmp     # 删除 Hadoop 临时文件
    sudo rm -r ./hadoop/logs/*   # 删除日志文件
    tar -zcf ~/hadoop.master.tar.gz ./hadoop   # 先压缩再复制
    cd ~
    scp ./hadoop.master.tar.gz Slave1:/home/hadoop
    ```
    在 Slave1 节点上执行：
    ```
    sudo rm -r /usr/local/hadoop    # 删掉旧的(如果存在)
    sudo tar -zxf ~/hadoop.master.tar.gz -C /usr/local
    sudo chown -R hadoop /usr/local/hadoop
    ```
    同样，如果有其他 Slave 节点，也要执行将 hadoop.master.tar.gz 传输到 Slave 节点、在 Slave 节点解压文件的操作。
    首次启动需要先在 Master 节点执行 NameNode 的格式化：
    ```
    hdfs namenode -format       # 首次运行需要执行初始化，之后不需要
    ```
3) 启动 hadoop
    启动需要在 Master 节点上进行:
    ```
    start-dfs.sh
    start-yarn.sh
    mr-jobhistory-daemon.sh start historyserver
    ```
4) 查看运行情况
    查看各个节点所启动的进程
    ```
    jps
    ```
    在 Master 节点上通过命令查看 DataNode 是否正常启动:
    ```
    hdfs dfsadmin -report
    ```
    通过 Web 页面看到查看 DataNode 和 NameNode 的状态：http://master:50070/
5) 关闭 Hadoop 集群
    在 Master 节点上执行:
    ```
    stop-yarn.sh
    stop-dfs.sh
    mr-jobhistory-daemon.sh stop historyserver
    ```
    此外，同伪分布式一样，也可以不启动 YARN，但要改掉 mapred-site.xml 的文件名。
    