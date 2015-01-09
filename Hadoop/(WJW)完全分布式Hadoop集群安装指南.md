# (WJW)高可用,完全分布式Hadoop集群HDFS和MapReduce安装配置指南    
>    
为了部署HA集群,应该准备以下事情:    
* namenode服务器:   运行namenode的服务器应该有相同的硬件配置.    
* journalnode服务器:运行的journalnode进程非常轻量,可以部署在其他的服务器上.注意:必须允许至少3个节点.当然可以运行更多,但是必须是奇数个,如3,5,7,9个等等.当运行N个节点时,系统可以容忍至少(N-1)/2个节点失败而不影响正常运行.     
在HA集群中,standby状态的namenode可以完成checkpoint操作,因此没必要配置Secondary namenode,CheckpointNode,BackupNode.如果真的配置了,还会报错.    

---

## [X] 安装环境:
+ 系统版本:CentOS 6.3 x86_64
+ JAVA版本:JDK-1.7.0_25
+ hadoop-2.2.0-src.tar.gz
+ 服务器列表:  
> 
`192.168.1.84 hadoop84` #**namenode1,resourcemanager**
> 
`192.168.1.85 hadoop85` #**namenode2,journalnode1,datanode1,nodemanager1**
> 
`192.168.1.86 hadoop86` #**journalnode2,datanode2,nodemanager2**
> 
`192.168.1.87 hadoop87` #**journalnode3,datanode3,nodemanager3**
     
+ JDK
>  
建议安装Sun的JDK1.7版本!
安装完毕并配置java环境变量,在/etc/profile末尾添加如下代码:  
export JAVA_HOME=/usr/java/default  
export PATH=$JAVA_HOME/bin:$PATH  
保存退出即可,然后执行`source /etc/profile`生效.在命令行执行java -version 如下代表JAVA安装成功.  

+ ssh
>  
需要配置各个节点的免密码登录!   
首先在自己机器上使用`ssh-keygen -t rsa`  
会要求输入密码(必须为空),回车几次,然后会在HOME目录下生成.ssh文件夹,  
里面有私钥和公钥,公钥为id_rsa.pub,(现在你需要将你的公钥拷贝到服务器上,如果你的系统有ssh-copy-id命令,拷贝会很简单:$ ssh-copy-id 用户名@服务器名)  
否则,你需要手动将你的私钥拷贝的服务器上的~/.ssh/authorized_keys文件中!  

+ NTP
>  
集群的时钟要保证基本的一致.稍有不一致是可以容忍的,但是很大的不一致会 造成奇怪的行为. 运行 NTP 或者其他什么东西来同步你的时间.    
如果你查询的时候或者是遇到奇怪的故障,可以检查一下系统时间是否正确!  
```bash
echo "server 192.168.0.2" >> /etc/ntp.conf  
chkconfig ntpd on  
service ntpd restart  
ntpq -p  
```

+ ulimit和nproc
>  
Hdaoop会在同一时间使用很多的文件句柄.大多数linux系统使用的默认值1024是不能满足的,修改`/etc/security/limits.conf`文件为:
```  
      *               soft    nproc   16384
      *               hard    nproc   16384  
      *               soft    nofile  65536  
      *               hard    nofile  65536  
```  

---

+ 修改 **192.168.1.84,192.168.1.85,192.168.1.86,192.168.1.87** 的`etc/hosts`文件
在文件最后添加:
>  
```
192.168.1.84 hadoop84
192.168.1.85 hadoop85
192.168.1.86 hadoop86
192.168.1.87 hadoop87
```

------

## [X] 编译hadoop
在hadoop84上操作
### [1] 拷贝`hadoop-2.2.0.tar.gz`到hadoop84的`/opt`目录下,然后执行:  
```bash
cd /opt
tar zxvf ./hadoop-2.2.0.tar.gz
```

### [2] YUM安装依赖库:  
```bash
yum install autoconfautomake libtool cmake zlib-devel
yum install ncurses-devel
yum install openssl-devel
yum install gcc*
```

### [3] 下载并安装配置:protobuf
```bash
cd /tmp
wget http://protobuf.googlecode.com/files/protobuf-2.5.0.tar.gz
tar -zxvf protobuf-2.5.0.tar.gz
cd protobuf-2.5.0
./configure
make
make install 
ldconfig
rm -rf /tmp/protobuf-2.5.0/
```

### [4] 下载并配置:findbugs
```bash
http://sourceforge.net/projects/findbugs/files/findbugs/2.0.2/findbugs-2.0.2.tar.gz/download
解压:tar -zxvf ./findbugs-2.0.2.tar.gz
配置环境变量FINDBUGS_HOME: export FINDBUGS_HOME=/path to your extract directory  #例如: export FINDBUGS_HOME=/opt/findbugs-2.0.2
```

### [5] 构建二进制版Hadoop
```
编辑`/opt/hadoop-2.2.0-src/hadoop-common-project/hadoop-auth/pom.xml`文件,添加上
    <dependency>
       <groupId>org.mortbay.jetty</groupId>
      <artifactId>jetty-util</artifactId>
      <scope>test</scope>
    </dependency>
编辑`/opt/hadoop-2.2.0-src/hadoop-common-project/hadoop-auth/pom.xml`文件,在<artifactId>maven-site-plugin</artifactId>后添加一行:<version>3.3</version>
编辑`/opt/hadoop-2.2.0-src/pom.xml`文件,把<artifactId>maven-site-plugin</artifactId>后面的:<version>3.0</version>改成:<version>3.3</version>

mvn package -Pdist,native,docs -DskipTests -Dtar
生成好的文件是:/opt/hadoop-2.2.0-src/hadoop-dist/target/hadoop-2.2.0.tar.gz
```

------

## [X] 安装Hadoop

### [1] 解压    
```
cp /opt/hadoop-2.2.0-src/hadoop-dist/target/hadoop-2.2.0.tar.gz /opt
cd /opt
tar -zxvf ./hadoop-2.2.0.tar.gz
mv hadoop-2.2.0  /opt/hadoop
```
>  
注意: 先在namenode服务器上都安装hadoop版本即可,datanode先不用安装,待会修改完配置后统一安装datanode!    

### [2] 配置环境变量    
修改`/opt/hadoop/libexec/hadoop-config.sh`,在最前面添加:
```bash
#->@wjw_add
export JAVA_HOME=/usr/java/default
export HADOOP_HOME=/opt/hadoop
export PATH=${PATH}:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin
export JAVA_LIBRARY_PATH=${HADOOP_HOME}/lib/native
export HADOOP_LOG_DIR=/opt/hadoop_data/logs
export YARN_LOG_DIR=${HADOOP_LOG_DIR}
#<-@wjw_add
```
>  
`hadoop-config.sh`会被其他所有的脚本来调用,可以把环境变量,JVM参数等都配置在这里!  

## [X] 配置Hadoop
先在`hdaoop84`上配置,我们需要修改如下几个地方:    

### [1] 修改`/opt/hadoop/etc/hadoop/core-site.xml`,内容为如下:
```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>fs.default.name</name>
    <value>hdfs://hadoop84:9000</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/opt/hadoop_data/tmp</value>
  </property>
</configuration>
```
>  
`fs.default.name` 因为我们会启动2个namenode,每个namenode的位置不一样,那么切换后,用户也要修改代码,很麻烦,    
因此`fs.default.name`使用一个逻辑路径,用户就可以不必担心namenode切换带来的路径不一致问题了.  
`hadoop.tmp.dir` 是hadoop文件系统依赖的基础配置,很多路径都依赖它.如果hdfs-site.xml中不配 置namenode和datanode的存放位置,默认就放在这个路径中.  

### [2] 修改`/opt/hadoop/etc/hadoop/hdfs-site.xml`,内容如下:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>/opt/hadoop_data/namenode</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/opt/hadoop_data/datanode</value>
  </property>  
  <property>
    <name>dfs.nameservices</name>
    <value>cluster1</value>
  </property>
  <property>
    <name>dfs.ha.namenodes.cluster1</name>
    <value>hadoop84,hadoop85</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.cluster1.hadoop84</name>
    <value>hadoop84:9000</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.cluster1.hadoop85</name>
    <value>hadoop85:9000</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.cluster1.hadoop84</name>
    <value>hadoop84:50070</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.cluster1.hadoop85</name>
    <value>hadoop85:50070</value>
  </property>
  <property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://hadoop85:8485;hadoop86:8485;hadoop87:8485/cluster1</value>
  </property>
  <property>
    <name>dfs.client.failover.proxy.provider.cluster1</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>
  <property>
    <name>dfs.ha.fencing.methods</name>
    <value>sshfence</value>
  </property>
  <property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/root/.ssh/id_rsa</value>
  </property>
  <property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/opt/hadoop_data/journal</value>
  </property>
</configuration>
```
>    
`dfs.replication` HDFS文件的副本数.如果你只有3个datanode,但是你却指定副本数为4,是不会生效的,因为每个datanode上只能存放一个副本.   
`dfs.namenode.name.dir` namenode的数据的存放位置    
`dfs.datanode.data.dir` namenode的数据的存放位置    
`dfs.nameservices` 命名空间的逻辑名称.如果使用HDFS Federation,可以配置多个命名空间的名称,使用逗号分开即可.    
`dfs.ha.namenodes.[nameservice ID]` 命名空间中所有namenode的唯一标示名称.可以配置多个,使用逗号分隔.该名称是可以让datanode知道每个集群的所有namenode.当前,每个集群最多只能配置两个namenode.    
`dfs.namenode.rpc-address.[nameservice ID].[namenode ID]` 每个namenode监听的RPC地址    
`dfs.namenode.http-address.[nameservice ID].[namenode ID]` 每个namenode监听的http地址    
`dfs.namenode.shared.edits.dir` 这是namenode读写JNS组的uri.通过这个uri,namenodes可以读写edit log内容.URI的格式"qjournal://host1:port1;host2:port2;host3:port3/journalId".    
 这里的host1,host2,host3指的是Journal Node的地址,这里必须是奇数个,至少3个;其中journalId是集群的唯一标识符,对于多个联邦命名空间,也使用同一个journalId.     
`dfs.client.failover.proxy.provider.[nameservice ID]` 这里配置HDFS客户端连接到Active namenode的一个java类.    
`dfs.ha.fencing.methods` 配置active namenode出错时的处理类.当active namenode出错时,一般需要关闭该进程.处理方式可以是ssh也可以是shell.推荐使用ssh!    
`fs.journalnode.edits.dir` 这是journalnode进程保持逻辑状态的路径.这是在linux服务器文件的绝对路径.    

### [3] 修改`/opt/hadoop/etc/hadoop/yarn-site.xml`,内容如下:
```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>hadoop84</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>
```
>    
`yarn.resourcemanager.hostname` 指的是运行ResourceManager机器所在的节点.    
`yarn.nodemanager.aux-services` 在hadoop2.2.0版本中是mapreduce_shuffle,一定要看清楚.

### [4] 修改`/opt/hadoop/etc/hadoop/mapred-site.xml.template`,内容如下:
先执行:`cp /opt/hadoop/etc/hadoop/mapred-site.xml.template /opt/hadoop/etc/hadoop/mapred-site.xml`    
```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```
>    
`mapreduce.framework.name` 指的是使用yarn运行mapreduce程序.    

### [5] 修改`/opt/hadoop/etc/hadoop/slaves`,内容如下:
```
hadoop85
hadoop86
hadoop87
```
>    
表示以上三个节点作为datanode和nodemanager节点.    

### [6] 把配置好的hadoop复制到其他节点:    
执行如下操作即可
```bash
ssh hadoop84
ssh hadoop85
ssh hadoop86
ssh hadoop87
scp -r /opt/hadoop/ root@hadoop85:/opt/
scp -r /opt/hadoop/ root@hadoop86:/opt/
scp -r /opt/hadoop/ root@hadoop87:/opt/
```
**自此整个集群基本搭建完毕,接下来就是启动hadoop集群了.**    

## [X] 执行命令启动HDFS和MapReduce集群:    
**以下命令严格注意执行顺序,不能颠倒!**    

### [1] 创建目录:    
在hadoop84执行命令:    
```bash
ssh hadoop84 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
ssh hadoop85 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
ssh hadoop86 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
ssh hadoop87 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
```

### [2] 启动JournalNode集群:    
在hadoop84执行命令:    
```bash
ssh hadoop85 '/opt/hadoop/sbin/hadoop-daemon.sh start journalnode'
ssh hadoop86 '/opt/hadoop/sbin/hadoop-daemon.sh start journalnode'
ssh hadoop87 '/opt/hadoop/sbin/hadoop-daemon.sh start journalnode'
```

### [3] 格式化第1个NameNode:
在hadoop84执行命令:
```bash
ssh hadoop84 '/opt/hadoop/bin/hdfs namenode -format -clusterId cluster1'
```

### [4] 启动第1个NameNode:
在hadoop84执行命令:
```bash
ssh hadoop84 '/opt/hadoop/sbin/hadoop-daemon.sh start namenode'
```

### [5] 格式化第2个NameNode:
在hadoop84执行命令:
```bash
ssh hadoop85 '/opt/hadoop/bin/hdfs namenode -bootstrapStandby'
```
### [6] 启动第2个NameNode:
在hadoop84执行命令:
```bash
ssh hadoop85 '/opt/hadoop/sbin/hadoop-daemon.sh start namenode'
```
>    
这时候,使用浏览器访问 http://hadoop84:50070 和 http://hadoop85:50070    
如果能够看到两个页面,证明NameNode启动成功了.这时,两个NameNode的状态都是standby.    

### [7] 转换active:
在hadoop84执行命令:
```bash
ssh hadoop84 '/opt/hadoop/bin/hdfs haadmin -failover --forceactive hadoop85 hadoop84'
```
>    
再使用浏览器访问 http://hadoop84:50070 和 http://hadoop85:50070    
会发现hadoop84节点变为active,hadoop85还是standby.

### [8] 启动DataNodes:
在hadoop84执行命令:
```bash
ssh hadoop84 '/opt/hadoop/sbin/hadoop-daemons.sh start datanode'
```
>    
会启动3个DataNode节点.    
这时候HA集群就启动了.

备注:
>    
你如果想实验一下NameNode切换,执行命令`hdfs haadmin -failover --forceactive hadoop84 hadoop85`
这时候观察hadoop84和hadoop85的状态,就会发现,已经改变了.    
如果向上传数据,还需要修改core-site.xml中的fs.default.name的值,改为hdfs://hadoop85:9000 才行.    

### [9] 启动MapReduce:
在hadoop84执行命令
```bash
ssh hadoop84 '/opt/hadoop/sbin/start-yarn.sh'
```
>  
用浏览器访问 http://hbase84:8088

------

# [X] 附录:
## [X] HA的问题:
>  
大家都知道在hadoop2中对HDFS的改进很大,实现了NameNode的HA;    
也增加了ResourceManager.但是ResourceManager也可以实现HA.    
你没看错,确实是ResourceManager的HA.注意是在Apache Hadoop 2.4.1版本中开始加入的,可不是任意一个版本.    


## [X] hadoop2的HA配置一键运行脚本startall.sh
```bash
#!/bin/sh

#synchronize all config files
ssh hadoop84 'scp /opt/hadoop/etc/hadoop/*  hadoop85:/opt/hadoop/etc/hadoop'
ssh hadoop84 'scp /opt/hadoop/etc/hadoop/*  hadoop86:/opt/hadoop/etc/hadoop'
ssh hadoop84 'scp /opt/hadoop/etc/hadoop/*  hadoop87:/opt/hadoop/etc/hadoop'
ssh hadoop84 'scp /opt/hadoop/libexec/*  hadoop85:/opt/hadoop/libexec'
ssh hadoop84 'scp /opt/hadoop/libexec/*  hadoop86:/opt/hadoop/libexec'
ssh hadoop84 'scp /opt/hadoop/libexec/*  hadoop87:/opt/hadoop/libexec'

#stop all daemons
ssh hadoop84 '/opt/hadoop/sbin/stop-all.sh'

#remove all files
ssh hadoop84 'rm -rf /opt/hadoop_data/'
ssh hadoop85 'rm -rf /opt/hadoop_data/'
ssh hadoop86 'rm -rf /opt/hadoop_data/'
ssh hadoop87 'rm -rf /opt/hadoop_data/'

ssh hadoop84 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
ssh hadoop85 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
ssh hadoop86 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
ssh hadoop87 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'

#start journalnodes cluster
ssh hadoop85 '/opt/hadoop/sbin/hadoop-daemon.sh start journalnode'
ssh hadoop86 '/opt/hadoop/sbin/hadoop-daemon.sh start journalnode'
ssh hadoop87 '/opt/hadoop/sbin/hadoop-daemon.sh start journalnode'

#format one namenode
ssh hadoop84 '/opt/hadoop/bin/hdfs namenode -format -clusterId cluster1'
ssh hadoop84 '/opt/hadoop/sbin/hadoop-daemon.sh start namenode'

#format another namenode
ssh hadoop85 '/opt/hadoop/bin/hdfs namenode -bootstrapStandby'
sleep 10
ssh hadoop85 '/opt/hadoop/sbin/hadoop-daemon.sh start namenode'
sleep 10

#trigger hadoop84 active
ssh hadoop84 '/opt/hadoop/bin/hdfs haadmin -failover --forceactive hadoop85 hadoop84'

#start all datanodes
ssh hadoop84 '/opt/hadoop/sbin/hadoop-daemons.sh start datanode'

#start MapReduce
sleep 10
ssh hadoop84 '/opt/hadoop/sbin/start-yarn.sh'
```
