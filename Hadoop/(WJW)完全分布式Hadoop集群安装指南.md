# (WJW)�߿���,��ȫ�ֲ�ʽHadoop��ȺHDFS��MapReduce��װ����ָ��    
>    
Ϊ�˲���HA��Ⱥ,Ӧ��׼����������:    
* namenode������:   ����namenode�ķ�����Ӧ������ͬ��Ӳ������.    
* journalnode������:���е�journalnode���̷ǳ�����,���Բ����������ķ�������.ע��:������������3���ڵ�.��Ȼ�������и���,���Ǳ�����������,��3,5,7,9���ȵ�.������N���ڵ�ʱ,ϵͳ������������(N-1)/2���ڵ�ʧ�ܶ���Ӱ����������.     
��HA��Ⱥ��,standby״̬��namenode�������checkpoint����,���û��Ҫ����Secondary namenode,CheckpointNode,BackupNode.������������,���ᱨ��.    

---

## [X] ��װ����:
+ ϵͳ�汾:CentOS 6.3 x86_64
+ JAVA�汾:JDK-1.7.0_25
+ hadoop-2.2.0-src.tar.gz
+ �������б�:  
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
���鰲װSun��JDK1.7�汾!
��װ��ϲ�����java��������,��/etc/profileĩβ������´���:  
export JAVA_HOME=/usr/java/default  
export PATH=$JAVA_HOME/bin:$PATH  
�����˳�����,Ȼ��ִ��`source /etc/profile`��Ч.��������ִ��java -version ���´���JAVA��װ�ɹ�.  

+ ssh
>  
��Ҫ���ø����ڵ���������¼!   
�������Լ�������ʹ��`ssh-keygen -t rsa`  
��Ҫ����������(����Ϊ��),�س�����,Ȼ�����HOMEĿ¼������.ssh�ļ���,  
������˽Կ�͹�Կ,��ԿΪid_rsa.pub,(��������Ҫ����Ĺ�Կ��������������,������ϵͳ��ssh-copy-id����,������ܼ�:$ ssh-copy-id �û���@��������)  
����,����Ҫ�ֶ������˽Կ�����ķ������ϵ�~/.ssh/authorized_keys�ļ���!  

+ NTP
>  
��Ⱥ��ʱ��Ҫ��֤������һ��.���в�һ���ǿ������̵�,���Ǻܴ�Ĳ�һ�»� �����ֵ���Ϊ. ���� NTP ��������ʲô������ͬ�����ʱ��.    
������ѯ��ʱ�������������ֵĹ���,���Լ��һ��ϵͳʱ���Ƿ���ȷ!  
```bash
echo "server 192.168.0.2" >> /etc/ntp.conf  
chkconfig ntpd on  
service ntpd restart  
ntpq -p  
```

+ ulimit��nproc
>  
Hdaoop����ͬһʱ��ʹ�úܶ���ļ����.�����linuxϵͳʹ�õ�Ĭ��ֵ1024�ǲ��������,�޸�`/etc/security/limits.conf`�ļ�Ϊ:
```  
      *               soft    nproc   16384
      *               hard    nproc   16384  
      *               soft    nofile  65536  
      *               hard    nofile  65536  
```  

---

+ �޸� **192.168.1.84,192.168.1.85,192.168.1.86,192.168.1.87** ��`etc/hosts`�ļ�
���ļ�������:
>  
```
192.168.1.84 hadoop84
192.168.1.85 hadoop85
192.168.1.86 hadoop86
192.168.1.87 hadoop87
```

------

## [X] ����hadoop
��hadoop84�ϲ���
### [1] ����`hadoop-2.2.0.tar.gz`��hadoop84��`/opt`Ŀ¼��,Ȼ��ִ��:  
```bash
cd /opt
tar zxvf ./hadoop-2.2.0.tar.gz
```

### [2] YUM��װ������:  
```bash
yum install autoconfautomake libtool cmake zlib-devel
yum install ncurses-devel
yum install openssl-devel
yum install gcc*
```

### [3] ���ز���װ����:protobuf
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

### [4] ���ز�����:findbugs
```bash
http://sourceforge.net/projects/findbugs/files/findbugs/2.0.2/findbugs-2.0.2.tar.gz/download
��ѹ:tar -zxvf ./findbugs-2.0.2.tar.gz
���û�������FINDBUGS_HOME: export FINDBUGS_HOME=/path to your extract directory  #����: export FINDBUGS_HOME=/opt/findbugs-2.0.2
```

### [5] ���������ư�Hadoop
```
�༭`/opt/hadoop-2.2.0-src/hadoop-common-project/hadoop-auth/pom.xml`�ļ�,�����
    <dependency>
       <groupId>org.mortbay.jetty</groupId>
      <artifactId>jetty-util</artifactId>
      <scope>test</scope>
    </dependency>
�༭`/opt/hadoop-2.2.0-src/hadoop-common-project/hadoop-auth/pom.xml`�ļ�,��<artifactId>maven-site-plugin</artifactId>�����һ��:<version>3.3</version>
�༭`/opt/hadoop-2.2.0-src/pom.xml`�ļ�,��<artifactId>maven-site-plugin</artifactId>�����:<version>3.0</version>�ĳ�:<version>3.3</version>

mvn package -Pdist,native,docs -DskipTests -Dtar
���ɺõ��ļ���:/opt/hadoop-2.2.0-src/hadoop-dist/target/hadoop-2.2.0.tar.gz
```

------

## [X] ��װHadoop

### [1] ��ѹ    
```
cp /opt/hadoop-2.2.0-src/hadoop-dist/target/hadoop-2.2.0.tar.gz /opt
cd /opt
tar -zxvf ./hadoop-2.2.0.tar.gz
mv hadoop-2.2.0  /opt/hadoop
```
>  
ע��: ����namenode�������϶���װhadoop�汾����,datanode�Ȳ��ð�װ,�����޸������ú�ͳһ��װdatanode!    

### [2] ���û�������    
�޸�`/opt/hadoop/libexec/hadoop-config.sh`,����ǰ�����:
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
`hadoop-config.sh`�ᱻ�������еĽű�������,���԰ѻ�������,JVM�����ȶ�����������!  

## [X] ����Hadoop
����`hdaoop84`������,������Ҫ�޸����¼����ط�:    

### [1] �޸�`/opt/hadoop/etc/hadoop/core-site.xml`,����Ϊ����:
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
`fs.default.name` ��Ϊ���ǻ�����2��namenode,ÿ��namenode��λ�ò�һ��,��ô�л���,�û�ҲҪ�޸Ĵ���,���鷳,    
���`fs.default.name`ʹ��һ���߼�·��,�û��Ϳ��Բ��ص���namenode�л�������·����һ��������.  
`hadoop.tmp.dir` ��hadoop�ļ�ϵͳ�����Ļ�������,�ܶ�·����������.���hdfs-site.xml�в��� ��namenode��datanode�Ĵ��λ��,Ĭ�Ͼͷ������·����.  

### [2] �޸�`/opt/hadoop/etc/hadoop/hdfs-site.xml`,��������:
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
`dfs.replication` HDFS�ļ��ĸ�����.�����ֻ��3��datanode,������ȴָ��������Ϊ4,�ǲ�����Ч��,��Ϊÿ��datanode��ֻ�ܴ��һ������.   
`dfs.namenode.name.dir` namenode�����ݵĴ��λ��    
`dfs.datanode.data.dir` namenode�����ݵĴ��λ��    
`dfs.nameservices` �����ռ���߼�����.���ʹ��HDFS Federation,�������ö�������ռ������,ʹ�ö��ŷֿ�����.    
`dfs.ha.namenodes.[nameservice ID]` �����ռ�������namenode��Ψһ��ʾ����.�������ö��,ʹ�ö��ŷָ�.�������ǿ�����datanode֪��ÿ����Ⱥ������namenode.��ǰ,ÿ����Ⱥ���ֻ����������namenode.    
`dfs.namenode.rpc-address.[nameservice ID].[namenode ID]` ÿ��namenode������RPC��ַ    
`dfs.namenode.http-address.[nameservice ID].[namenode ID]` ÿ��namenode������http��ַ    
`dfs.namenode.shared.edits.dir` ����namenode��дJNS���uri.ͨ�����uri,namenodes���Զ�дedit log����.URI�ĸ�ʽ"qjournal://host1:port1;host2:port2;host3:port3/journalId".    
 �����host1,host2,host3ָ����Journal Node�ĵ�ַ,���������������,����3��;����journalId�Ǽ�Ⱥ��Ψһ��ʶ��,���ڶ�����������ռ�,Ҳʹ��ͬһ��journalId.     
`dfs.client.failover.proxy.provider.[nameservice ID]` ��������HDFS�ͻ������ӵ�Active namenode��һ��java��.    
`dfs.ha.fencing.methods` ����active namenode����ʱ�Ĵ�����.��active namenode����ʱ,һ����Ҫ�رոý���.����ʽ������sshҲ������shell.�Ƽ�ʹ��ssh!    
`fs.journalnode.edits.dir` ����journalnode���̱����߼�״̬��·��.������linux�������ļ��ľ���·��.    

### [3] �޸�`/opt/hadoop/etc/hadoop/yarn-site.xml`,��������:
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
`yarn.resourcemanager.hostname` ָ��������ResourceManager�������ڵĽڵ�.    
`yarn.nodemanager.aux-services` ��hadoop2.2.0�汾����mapreduce_shuffle,һ��Ҫ�����.

### [4] �޸�`/opt/hadoop/etc/hadoop/mapred-site.xml.template`,��������:
��ִ��:`cp /opt/hadoop/etc/hadoop/mapred-site.xml.template /opt/hadoop/etc/hadoop/mapred-site.xml`    
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
`mapreduce.framework.name` ָ����ʹ��yarn����mapreduce����.    

### [5] �޸�`/opt/hadoop/etc/hadoop/slaves`,��������:
```
hadoop85
hadoop86
hadoop87
```
>    
��ʾ���������ڵ���Ϊdatanode��nodemanager�ڵ�.    

### [6] �����úõ�hadoop���Ƶ������ڵ�:    
ִ�����²�������
```bash
ssh hadoop84
ssh hadoop85
ssh hadoop86
ssh hadoop87
scp -r /opt/hadoop/ root@hadoop85:/opt/
scp -r /opt/hadoop/ root@hadoop86:/opt/
scp -r /opt/hadoop/ root@hadoop87:/opt/
```
**�Դ�������Ⱥ��������,��������������hadoop��Ⱥ��.**    

## [X] ִ����������HDFS��MapReduce��Ⱥ:    
**���������ϸ�ע��ִ��˳��,���ܵߵ�!**    

### [1] ����Ŀ¼:    
��hadoop84ִ������:    
```bash
ssh hadoop84 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
ssh hadoop85 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
ssh hadoop86 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
ssh hadoop87 'mkdir -p /opt/hadoop_data/tmp /opt/hadoop_data/namenode /opt/hadoop_data/datanode /opt/hadoop_data/journal'
```

### [2] ����JournalNode��Ⱥ:    
��hadoop84ִ������:    
```bash
ssh hadoop85 '/opt/hadoop/sbin/hadoop-daemon.sh start journalnode'
ssh hadoop86 '/opt/hadoop/sbin/hadoop-daemon.sh start journalnode'
ssh hadoop87 '/opt/hadoop/sbin/hadoop-daemon.sh start journalnode'
```

### [3] ��ʽ����1��NameNode:
��hadoop84ִ������:
```bash
ssh hadoop84 '/opt/hadoop/bin/hdfs namenode -format -clusterId cluster1'
```

### [4] ������1��NameNode:
��hadoop84ִ������:
```bash
ssh hadoop84 '/opt/hadoop/sbin/hadoop-daemon.sh start namenode'
```

### [5] ��ʽ����2��NameNode:
��hadoop84ִ������:
```bash
ssh hadoop85 '/opt/hadoop/bin/hdfs namenode -bootstrapStandby'
```
### [6] ������2��NameNode:
��hadoop84ִ������:
```bash
ssh hadoop85 '/opt/hadoop/sbin/hadoop-daemon.sh start namenode'
```
>    
��ʱ��,ʹ����������� http://hadoop84:50070 �� http://hadoop85:50070    
����ܹ���������ҳ��,֤��NameNode�����ɹ���.��ʱ,����NameNode��״̬����standby.    

### [7] ת��active:
��hadoop84ִ������:
```bash
ssh hadoop84 '/opt/hadoop/bin/hdfs haadmin -failover --forceactive hadoop85 hadoop84'
```
>    
��ʹ����������� http://hadoop84:50070 �� http://hadoop85:50070    
�ᷢ��hadoop84�ڵ��Ϊactive,hadoop85����standby.

### [8] ����DataNodes:
��hadoop84ִ������:
```bash
ssh hadoop84 '/opt/hadoop/sbin/hadoop-daemons.sh start datanode'
```
>    
������3��DataNode�ڵ�.    
��ʱ��HA��Ⱥ��������.

��ע:
>    
�������ʵ��һ��NameNode�л�,ִ������`hdfs haadmin -failover --forceactive hadoop84 hadoop85`
��ʱ��۲�hadoop84��hadoop85��״̬,�ͻᷢ��,�Ѿ��ı���.    
������ϴ�����,����Ҫ�޸�core-site.xml�е�fs.default.name��ֵ,��Ϊhdfs://hadoop85:9000 ����.    

### [9] ����MapReduce:
��hadoop84ִ������
```bash
ssh hadoop84 '/opt/hadoop/sbin/start-yarn.sh'
```
>  
����������� http://hbase84:8088

------

# [X] ��¼:
## [X] HA������:
>  
��Ҷ�֪����hadoop2�ж�HDFS�ĸĽ��ܴ�,ʵ����NameNode��HA;    
Ҳ������ResourceManager.����ResourceManagerҲ����ʵ��HA.    
��û����,ȷʵ��ResourceManager��HA.ע������Apache Hadoop 2.4.1�汾�п�ʼ�����,�ɲ�������һ���汾.    


## [X] hadoop2��HA����һ�����нű�startall.sh
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
