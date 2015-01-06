# (WJW)�����ⲿZooKeeper��GlusterFS��Ϊ�ֲ�ʽ�ļ�ϵͳ����ȫ�ֲ�ʽHBase��Ⱥ��װָ��

---

## [X] ǰ������
+ �������б�:  
> 
`192.168.1.84 hbase84` #**hbase-master**
> 
`192.168.1.85 hbase85` #**hbase-regionserver,zookeeper**
> 
`192.168.1.86 hbase86` #**hbase-regionserver,zookeeper**
> 
`192.168.1.87 hbase87` #**hbase-regionserver,zookeeper**
     
+ JDK
>  
���鰲װSun��JDK1.7�汾!

+ ssh
>  
��Ҫ���ø����ڵ���������¼!

+ NTP
>  
��Ⱥ��ʱ��Ҫ��֤������һ��.���в�һ���ǿ������̵�,���Ǻܴ�Ĳ�һ�»� �����ֵ���Ϊ. ���� NTP ��������ʲô������ͬ�����ʱ��.    
������ѯ��ʱ�������������ֵĹ���,���Լ��һ��ϵͳʱ���Ƿ���ȷ!

+ ulimit��nproc
>  
HBase�����ݿ�,����ͬһʱ��ʹ�úܶ���ļ����.�����linuxϵͳʹ�õ�Ĭ��ֵ1024�ǲ��������,�޸�`/etc/security/limits.conf`�ļ�Ϊ:
```  
      *               soft    nproc   16384
      *               hard    nproc   16384  
      *               soft    nofile  65536  
      *               hard    nofile  65536  
```  

+ ZooKeeper  
>  
1. �Ȱ�ZooKeeper��װ��`hbase85,hbase86,hbase87`3���ڵ���    
2. ����`hbase85,hbase86,hbase87`�ϵ�ZooKeeper!  
>  >  `/opt/app/zookeeper/bin/zkServer.sh start`

+ HBase����Ҫһ���������еķֲ�ʽ�ļ�ϵͳ:`HDFS`,����`GlusterFS`
>  
��ָ��ʹ��`GlusterFS`��Ϊ�ֲ�ʽ�ļ�ϵͳ!  
ÿ���ڵ�Ĺ���Ŀ¼Ϊ:`/mnt/gfs_v3/hbase`  
GlusterFS��Scale-Out�洢�������Gluster�ĺ���,����һ����Դ�ķֲ�ʽ�ļ�ϵͳ,����ǿ��ĺ�����չ����,ͨ����չ�ܹ�֧����PB�洢�����ʹ�����ǧ�ͻ���.  
GlusterFS����TCP/IP��InfiniBand RDMA���罫����ֲ��Ĵ洢��Դ�ۼ���һ��,ʹ�õ�һȫ�������ռ�����������.  
GlusterFS���ڿɶѵ����û��ռ����,��Ϊ���ֲ�ͬ�����ݸ����ṩ���������.  

---

## [X]  �޸� **192.168.1.84,192.168.1.85,192.168.1.86,192.168.1.87** ��`etc/hosts`�ļ�,���ļ�������:
```
192.168.1.84 hbase84
192.168.1.85 hbase85
192.168.1.86 hbase86
192.168.1.87 hbase87
```

## [X]  ����`hbase-0.98.8-hadoop2-bin.tar.gz`�������ڵ��`/opt`Ŀ¼��,Ȼ��ִ��:  
```bash
cd /opt
tar zxvf ./hbase-0.98.8-hadoop2-bin.tar.gz
```

## [X]  Դ����밲װHadoop��native��:  
```
#YUM��װ������
yum install autoconfautomake libtool cmake
yum install ncurses-devel
yum install openssl-devel
yum install gcc*

#Download and install protobuf
wget http://protobuf.googlecode.com/files/protobuf-2.5.0.tar.gz /tmp
tar -xvf protobuf-2.5.0.tar.gz
cd /tmp/protobuf-2.5.0
./configure
make
make install 
ldconfig

#Install cmake
yum install cmake

#Build native libraries using maven
mvn package -Pdist,native -DskipTests -Dtar
```
��hadoop2.2.0���`lib/native`Ŀ¼�µ��ļ����Ƶ�hbase��`lib/native`Ŀ¼��,Ȼ��ִ��:  
>  
cd /opt/hbase-0.98.8-hadoop2/lib/native  
ln -fs libhadoop.so.1.0.0 libhadoop.so  
ln -fs libhdfs.so.0.0.0 libhdfs.so  


## [X]  �޸�`conf/hbase-env.sh`�ļ�,���ļ�������:
```
export JAVA_HOME=/usr/java/default
export PATH=$JAVA_HOME/bin:$PATH

export HBASE_PID_DIR=/var/hadoop/pids
export JAVA_LIBRARY_PATH=${HBASE_HOME}/lib/native
export HBASE_MANAGES_ZK=false
export HBASE_HEAPSIZE=3000
```

## [X]  �޸�`conf/hbase-site.xml`�ļ�,�ĳ�:
```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.master.port</name>
    <value>60000</value>
  </property>

  <property>
    <name>hbase.rootdir</name>
    <value>file:///mnt/gfs_v3/hbase</value>
    <!-- <value>hdfs://m1:9000/hbase</value>  -->
  </property>
  
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>192.168.1.85,192.168.1.86,192.168.1.87</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.clientPort</name>
    <value>2181</value>
  </property>
  
  <!-- �Ż�����  -->
  <property>
    <name>hbase.regionserver.handler.count</name>
    <value>50</value>
  </property>
  <property>
    <name>hbase.hregion.max.filesize</name>
    <value>268435456</value>
  </property>
  
</configuration>
```

## [X]  �޸�`conf/regionservers�ļ�`,����HBase��Ⱥ�еĴӽ��Region Server,������ʾ:
``` 
hbase85
hbase86
hbase87
```

## [X]  Ϊ�ű��ļ���ӿ�ִ��Ȩ��
```bash
chmod -R +x /opt/hbase-0.98.8-hadoop2/bin/*
chmod -R +x /opt/hbase-0.98.8-hadoop2/conf/*.sh
```

## [X]  ����hbase��Ⱥ  
��master�ڵ���ִ��:  
```bash
/opt/hbase-0.98.8-hadoop2/bin/start-hbase.sh
```

## [X]  ֹͣhbase��Ⱥ
��master�ڵ���ִ��:  
```bash
/opt/hbase-0.98.8-hadoop2/bin/stop-hbase.sh
```

# [X] ��¼:

## �ű�ʹ��С��: 
1. ������Ⱥ,`start-hbase.sh`   
2. �رռ�Ⱥ,`stop-hbase.sh`   
3. ����/�ر����е�**regionserver��zookeeper**, `hbase-daemons.sh start/stop regionserver/zookeeper`   
4. ����/�رյ���**regionserver��zookeeper**, `hbase-daemon.sh start/stop regionserver/zookeeper`   
5. ����/�ر�**master**, `hbase-daemon.sh start/stop master`, �Ƿ��Ϊactive masterȡ���ڵ�ǰ�Ƿ���active master   
6. rolling-restart.sh �������������������� 
7. graceful_stop.sh move�������ϵ�����region��,��stop/restart�÷�����,�����������а汾�������� 

����ϸ��: 
>  
1. hbase-daemon.sh start master �� hbase-daemon.sh start master --backup,��2�����������һ����,�Ƿ��Ϊbackup��active����master���ڲ��߼������Ƶ�   
2. stop-hbase.sh �������hbase-daemons.sh stop regionserver ���ر�regionserver, ���ǻ����hbase-daemons.sh stop zookeeper/master-backup���ر�zk��backup master,�ر�regionserverʵ�ʵ��õ���hbaseAdmin��shutdown�ӿ�   
3. ͨ��$HBASE_HOME/bin/hbase stop master�رյ���������Ⱥ���ǵ���master,ֻ�رյ���master�Ļ�ʹ��$HBASE_HOME/bin/hbase-daemon.sh stop master   
4. $HBASE_HOME/bin/hbase stop regionserver/zookeeper ������ô��,����Ҳ�����,Ҳû��·��������������,���ǿ���ͨ��$HBASE_HOME/bin/hbase start regionserver/zookeeper ������rs����zk,hbase-daemon.sh���õľ����������  

----------

## HTableһЩ��������
### Row key
> + 
������, HBase��֧��������ѯ��Order by�Ȳ�ѯ,��ȡ��¼ֻ�ܰ�Row key(����range)��ȫ��ɨ��,���Row key��Ҫ����ҵ���������������洢��������(Table��Row key�ֵ���������1,10,100,11,2)�������.
> + 
����ʹ�õ��������ֻ�ʱ����Ϊrowkey.
> + 
���rowkey������,�ö����Ƶķ�ʽ����string���洢����Լ�ռ�
> + 
����Ŀ���rowkey�ĳ���,�����ܶ�,��Ϊrowkey������Ҳ�����ÿ��Cell��.
> + 
�����Ҫ����Ԥ����Ϊ���region��,����Զ�����ѵĹ���.

### Column Family(����)
> + 
 �ڱ���ʱ����,ÿ��Column FamilyΪһ���洢��Ԫ.�������������һ��HBase��blog,�ñ�����������:article��author.    
> +  
 �дؾ�����,��ò�����3��.��Ϊÿ���д��Ǵ���һ��������HFile���,flush��compaction�����������һ��Region���е�,��һ���дص����ݺܶ���Ҫflush��ʱ��,�����дؼ�ʹ���ݺ���Ҳ��Ҫflush,�����Ͳ����Ĵ�������Ҫ��io����.
> +  
 �ڶ��дص������,ע����д����ݵ�������Ҫһ��.��������дص����������̫��,��ʹ�������ٵ��дص�����ɨ��Ч�ʵ���.
> +  
 ��������ѯ�Ͳ�������ѯ�����ݷŵ���ͬ���д�.
> +  
 ��Ϊ�дغ��е����ֻ����HBase��ÿ��Cell��,�������ǵ�����Ӧ�þ����ܵĶ�.����,��f:q����mycolumnfamily:mycolumnqualifier

### Column(��)
> +  
HBase��ÿ���ж�����һ������,��������Ϊǰ׺,����article:title��article:content����article����,author:name��author:nickname����author����.
> + 
Column���ô�����ʱ���弴���Զ�̬����,ͬһColumn Family��Columns��Ⱥ����һ���洢��Ԫ��,����Column key����,������ʱӦ��������ͬI/O���Ե�Column�����һ��Column Family�����������.ͬʱ������Ҫע�����:������ǿ������Ӻ�ɾ����,������ǵĴ�ͳ���ݿ�ܴ������.�������ʺϷǽṹ������.

### Timestamp
> + 
HBaseͨ��row��columnȷ��һ������,������ݵ�ֵ�����ж���汾,��ͬ�汾��ֵ����ʱ�䵹������,�����µ�����������ǰ��,��ѯʱĬ�Ϸ������°汾.��������row key=1��author:nicknameֵ�������汾,�ֱ�Ϊ1317180070811��Ӧ��"һҶ�ɽ�"��1317180718830��Ӧ��"yedu"(��Ӧ��ʵ��ҵ��������Ϊ��ĳʱ���޸���nicknameΪyedu,����ֵ��Ȼ����).TimestampĬ��Ϊϵͳ��ǰʱ��(��ȷ������),Ҳ������д������ʱָ����ֵ.

### Value
> + 
ÿ��ֵͨ��4����Ψһ����,tableName+RowKey+ColumnKey+Timestamp=>value,����������{tableName='blog',RowKey='1',ColumnName='author:nickname',Timestamp=' 1317180718830'}��������Ψһֵ��"yedu".

### �洢����
>  
`TableName` ���ַ���  
`RowKey`��`ColumnName`�Ƕ�����ֵ(Java ���� byte[])  
`Timestamp`��һ�� 64 λ����(Java ���� long) 
`value`��һ���ֽ�����(Java���� byte[]).    

---    
## HBase�����Ż�
### zookeeper.session.timeout
> 
**Ĭ��ֵ**:3����(180000ms)
> 
**˵��**:RegionServer��Zookeeper������ӳ�ʱʱ��.����ʱʱ�䵽��,ReigonServer�ᱻZookeeper��RS��Ⱥ�嵥���Ƴ�,HMaster�յ��Ƴ�֪ͨ��,�����̨server�����regions����balance,����������RegionServer�ӹ�.
> 
**����**:���timeout������RegionServer�Ƿ��ܹ���ʱ��failover.���ó�1���ӻ����,���Լ�����ȴ���ʱ�����ӳ���failoverʱ��.
������Ҫע�����,����һЩOnlineӦ��,RegionServer��崻����ָ�ʱ�䱾��ͺ̵ܶ�(��������,crash�ȹ���,��ά�ɿ��ٽ���),�������timeoutʱ��,������ò���ʧ.��Ϊ��ReigonServer����ʽ��RS��Ⱥ���Ƴ�ʱ,HMaster�Ϳ�ʼ��balance��(������RS���ݹ��ϻ�����¼��WAL��־���лָ�).�����ϵ�RS���˹�����ָ���,���balance�����Ǻ��������,������ʹ���ز�����,��RS�������ฺ��.�ر�����Щ�̶�����regions�ĳ���.

### hbase.regionserver.handler.count
> 
**Ĭ��ֵ**:10
> 
**˵��**:RegionServer��������IO�߳���.
> 
**����**:
��������ĵ������ڴ�ϢϢ���.
���ٵ�IO�߳�,�����ڴ����������ڴ����Ľϸߵ�Big PUT����(����������PUT�������˽ϴ�cache��scan,������Big PUT)��ReigonServer���ڴ�ȽϽ��ŵĳ���.
�϶��IO�߳�,�����ڵ��������ڴ����ĵ�,TPSҪ��ǳ��ߵĳ���.���ø�ֵ��ʱ��,�Լ���ڴ�Ϊ��Ҫ�ο�.
������Ҫע��������server��region��������,��������������һ��region��,����ٳ���memstore����flush���µĶ�д����Ӱ��ȫ��TPS,����IO�߳���Խ��Խ��.
ѹ��ʱ,����[Enabling RPC-level logging](http://hbase.apache.org/book.html#rpc.logging),  
����ͬʱ���ÿ��������ڴ����ĺ�GC��״��,���ͨ�����ѹ�������������IO�߳���.
������һ������?[Hadoop and HBase Optimization for Read Intensive Search Applications](http://software.intel.com/en-us/articles/hadoop-and-hbase-optimization-for-read-intensive-search-applications/),������SSD�Ļ���������IO�߳���Ϊ100,�����ο�.

### hbase.hregion.max.filesize
> 
**Ĭ��ֵ**:256M
> 
**˵��**:�ڵ�ǰReigonServer�ϵ���Reigon�����洢�ռ�,����Region������ֵʱ,���Region�ᱻ�Զ�split�ɸ�С��region.
> 
**����**:
Сregion��split��compaction�Ѻ�,��Ϊ���region��compactСregion���storefile�ٶȺܿ�,�ڴ�ռ�õ�.ȱ����split��compaction���Ƶ��.�ر��������϶��Сregion��ͣ��split, compaction,�ᵼ�¼�Ⱥ��Ӧʱ�䲨���ܴ�,region����̫�಻���������ϴ����鷳,����������һЩHbase��bug.һ��512���µĶ���Сregion.  
��region,��̫�ʺϾ���split��compaction,��Ϊ��һ��compact��split������ϳ�ʱ���ͣ��,��Ӧ�õĶ�д���ܳ���ǳ���.����,��region��ζ�Žϴ��storefile,compactionʱ���ڴ�Ҳ��һ����ս.
��Ȼ,��regionҲ��������֮��.������Ӧ�ó�����,ĳ��ʱ���ķ������ϵ�,��ô�ڴ�ʱ��compact��split,����˳�����split��compaction,���ܱ�֤�������ʱ��ƽ�ȵĶ�д����.  
��Ȼsplit��compaction���Ӱ������,��û�а취ȥ��?
compaction���޷������,split���ǿ��Դ��Զ�����Ϊ�ֶ�.
ֻҪͨ�����������ֵ����ĳ�����Ѵﵽ��ֵ,����100G,�Ϳ��Լ�ӽ����Զ�split(RegionServer�����δ����100G��region��split).
�����RegionSplitter�������,����Ҫsplitʱ,�ֶ�split.
�ֶ�split������Ժ��ȶ����ϱ����Զ�splitҪ�ߺܶ�,�෴,����ɱ����Ӳ���,�Ƚ��Ƽ�onlineʵʱϵͳʹ��.
�ڴ淽��,Сregion������memstore�Ĵ�Сֵ�ϱȽ����,��region������С������,����ᵼ��flushʱapp��IO wait����,��С����store file����Ӱ�������.

### hbase.regionserver.global.memstore.upperLimit/lowerLimit
> 
Ĭ��ֵ:0.4/0.35
> 
**upperlimit˵��**:hbase.hregion.memstore.flush.size ��������������ǵ�����Region�����е�memstore��С�ܺͳ���ָ��ֵʱ,flush��region������memstore.RegionServer��flush��ͨ�����������һ������,ģ����������ģʽ���첽�����.���������һ������,����������������,����������ѹ����ʱ,���ܻᵼ���ڴ涸��,�������Ǵ���OOM.
��������������Ƿ�ֹ�ڴ�ռ�ù���,��ReigonServer������region��memstores��ռ���ڴ��ܺʹﵽheap��40%ʱ,HBase��ǿ��block���еĸ��²�flush��Щregion���ͷ�����memstoreռ�õ��ڴ�.
>  
**lowerLimit˵��**: ͬupperLimit,ֻ����lowerLimit������region��memstores��ռ���ڴ�ﵽHeap��35%ʱ,��flush���е�memstore.������һ��memstore�ڴ�ռ������region,������flush,��ʱд���»��ǻᱻblock.lowerLimit����һ��������regionǿ��flush�������ܽ���ǰ�Ĳ��ȴ�ʩ.����־��,����Ϊ "** Flush thread woke up with memory above low water."  
>  
**����**:����һ��Heap�ڴ汣������,Ĭ��ֵ�Ѿ������ô��������.
����������Ӱ���д,���д��ѹ�����¾������������ֵ,���С������hfile.block.cache.size����÷�ֵ,����Heap�����϶�ʱ,���޸Ķ������С.
����ڸ�ѹ�����,Ҳû���������ֵ,��ô�������ʵ���С�����ֵ����ѹ��,ȷ������������Ҫ̫��,Ȼ���н϶�Heap������ʱ��,����hfile.block.cache.size��߶�����.
����һ�ֿ�������?hbase.hregion.memstore.flush.size���ֲ���,��RSά���˹����region,Ҫ֪�� region����ֱ��Ӱ��ռ���ڴ�Ĵ�С.

### hfile.block.cache.size
> 
**Ĭ��ֵ**:0.2
> 
**˵��**:storefile�Ķ�����ռ��Heap�Ĵ�С�ٷֱ�,0.2��ʾ20%.��ֱֵ��Ӱ�����ݶ�������.
> 
**����**:��Ȼ��Խ��Խ��,���д�ȶ��ٺܶ�,����0.4-0.5Ҳû����.�����д�Ͼ���,0.3����.���д�ȶ���,����Ĭ�ϰ�.�������ֵ��ʱ��,��ͬʱҪ�ο�?hbase.regionserver.global.memstore.upperLimit?,��ֵ��memstoreռheap�����ٷֱ�,��������һ��Ӱ���,һ��Ӱ��д.�����ֵ����������80-90%,����OOM�ķ���,��������.

### hbase.hstore.blockingStoreFiles
> 
**Ĭ��ֵ**:7
> 
**˵��**:��flushʱ,��һ��region�е�Store(Coulmn Family)���г���7��storefileʱ,��block���е�д�������compaction,�Լ���storefile����.
> 
**����**:blockд���������Ӱ�쵱ǰregionServer����Ӧʱ��,�������storefileҲ��Ӱ�������.��ʵ��Ӧ������,Ϊ�˻�ȡ��ƽ������Ӧʱ��,�ɽ�ֵ��Ϊ���޴�.�����������Ӧʱ����ֽϴ�Ĳ��岨��,��ôĬ�ϻ������������������.

### hbase.hregion.memstore.block.multiplier
> 
**Ĭ��ֵ**:2
> 
**˵��**:��һ��region���memstoreռ���ڴ��С����hbase.hregion.memstore.flush.size�����Ĵ�Сʱ,block��region����������,����flush,�ͷ��ڴ�.
��Ȼ����������region��ռ�õ�memstores���ڴ��С,����64M,������һ��,�����63.9M��ʱ��,��Put��һ��200M������,��ʱmemstore�Ĵ�С��˲�䱩�ǵ�����Ԥ�ڵ�hbase.hregion.memstore.flush.size�ļ���.��������������ǵ�memstore�Ĵ�С��������hbase.hregion.memstore.flush.size 2��ʱ,block��������,���Ʒ��ս�һ������.
> 
**����**: ���������Ĭ��ֵ���ǱȽϿ��׵�.�����Ԥ���������Ӧ�ó���(�������쳣)�������ͻ��д��д�����ɿ�,��ô����Ĭ��ֵ����.������������,���д�������ͻᾭ�������������ļ���,��ô��Ӧ�õ������������������������ֵ,����hfile.block.cache.size��hbase.regionserver.global.memstore.upperLimit/lowerLimit,��Ԥ�������ڴ�,��ֹHBase server OOM.

### hbase.hregion.memstore.mslab.enabled
> 
**Ĭ��ֵ**:true
> 
**˵��**:�������ڴ���Ƭ���µ�Full GC,�����������.
> 
**����**:��� [http://kenwublog.com/avoid-full-gc-in-hbase-using-arena-allocation](http://kenwublog.com/avoid-full-gc-in-hbase-using-arena-allocation)


---  


