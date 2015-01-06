# (WJW)基于外部ZooKeeper的GlusterFS作为分布式文件系统的完全分布式HBase集群安装指南

---

## [X] 前提条件
+ 服务器列表:  
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
建议安装Sun的JDK1.7版本!

+ ssh
>  
需要配置各个节点的免密码登录!

+ NTP
>  
集群的时钟要保证基本的一致.稍有不一致是可以容忍的,但是很大的不一致会 造成奇怪的行为. 运行 NTP 或者其他什么东西来同步你的时间.    
如果你查询的时候或者是遇到奇怪的故障,可以检查一下系统时间是否正确!

+ ulimit和nproc
>  
HBase是数据库,会在同一时间使用很多的文件句柄.大多数linux系统使用的默认值1024是不能满足的,修改`/etc/security/limits.conf`文件为:
```  
      *               soft    nproc   16384
      *               hard    nproc   16384  
      *               soft    nofile  65536  
      *               hard    nofile  65536  
```  

+ ZooKeeper  
>  
1. 先把ZooKeeper安装在`hbase85,hbase86,hbase87`3个节点上    
2. 启动`hbase85,hbase86,hbase87`上的ZooKeeper!  
>  >  `/opt/app/zookeeper/bin/zkServer.sh start`

+ HBase还需要一个正在运行的分布式文件系统:`HDFS`,或者`GlusterFS`
>  
本指南使用`GlusterFS`作为分布式文件系统!  
每个节点的挂载目录为:`/mnt/gfs_v3/hbase`  
GlusterFS是Scale-Out存储解决方案Gluster的核心,它是一个开源的分布式文件系统,具有强大的横向扩展能力,通过扩展能够支持数PB存储容量和处理数千客户端.  
GlusterFS借助TCP/IP或InfiniBand RDMA网络将物理分布的存储资源聚集在一起,使用单一全局命名空间来管理数据.  
GlusterFS基于可堆叠的用户空间设计,可为各种不同的数据负载提供优异的性能.  

---

## [X]  修改 **192.168.1.84,192.168.1.85,192.168.1.86,192.168.1.87** 的`etc/hosts`文件,在文件最后添加:
```
192.168.1.84 hbase84
192.168.1.85 hbase85
192.168.1.86 hbase86
192.168.1.87 hbase87
```

## [X]  拷贝`hbase-0.98.8-hadoop2-bin.tar.gz`到各个节点的`/opt`目录下,然后执行:  
```bash
cd /opt
tar zxvf ./hbase-0.98.8-hadoop2-bin.tar.gz
```

## [X]  源码编译安装Hadoop的native库:  
```
#YUM安装依赖库
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
把hadoop2.2.0里的`lib/native`目录下的文件复制到hbase的`lib/native`目录下,然后执行:  
>  
cd /opt/hbase-0.98.8-hadoop2/lib/native  
ln -fs libhadoop.so.1.0.0 libhadoop.so  
ln -fs libhdfs.so.0.0.0 libhdfs.so  


## [X]  修改`conf/hbase-env.sh`文件,在文件最后添加:
```
export JAVA_HOME=/usr/java/default
export PATH=$JAVA_HOME/bin:$PATH

export HBASE_PID_DIR=/var/hadoop/pids
export JAVA_LIBRARY_PATH=${HBASE_HOME}/lib/native
export HBASE_MANAGES_ZK=false
export HBASE_HEAPSIZE=3000
```

## [X]  修改`conf/hbase-site.xml`文件,改成:
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
  
  <!-- 优化参数  -->
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

## [X]  修改`conf/regionservers文件`,配置HBase集群中的从结点Region Server,如下所示:
``` 
hbase85
hbase86
hbase87
```

## [X]  为脚本文件添加可执行权限
```bash
chmod -R +x /opt/hbase-0.98.8-hadoop2/bin/*
chmod -R +x /opt/hbase-0.98.8-hadoop2/conf/*.sh
```

## [X]  启动hbase集群  
在master节点上执行:  
```bash
/opt/hbase-0.98.8-hadoop2/bin/start-hbase.sh
```

## [X]  停止hbase集群
在master节点上执行:  
```bash
/opt/hbase-0.98.8-hadoop2/bin/stop-hbase.sh
```

# [X] 附录:

## 脚本使用小结: 
1. 开启集群,`start-hbase.sh`   
2. 关闭集群,`stop-hbase.sh`   
3. 开启/关闭所有的**regionserver、zookeeper**, `hbase-daemons.sh start/stop regionserver/zookeeper`   
4. 开启/关闭单个**regionserver、zookeeper**, `hbase-daemon.sh start/stop regionserver/zookeeper`   
5. 开启/关闭**master**, `hbase-daemon.sh start/stop master`, 是否成为active master取决于当前是否有active master   
6. rolling-restart.sh 可以用来挨个滚动重启 
7. graceful_stop.sh move服务器上的所有region后,再stop/restart该服务器,可以用来进行版本的热升级 

几个细节: 
>  
1. hbase-daemon.sh start master 与 hbase-daemon.sh start master --backup,这2个命令的作用一样的,是否成为backup或active是由master的内部逻辑来控制的   
2. stop-hbase.sh 不会调用hbase-daemons.sh stop regionserver 来关闭regionserver, 但是会调用hbase-daemons.sh stop zookeeper/master-backup来关闭zk和backup master,关闭regionserver实际调用的是hbaseAdmin的shutdown接口   
3. 通过$HBASE_HOME/bin/hbase stop master关闭的是整个集群而非单个master,只关闭单个master的话使用$HBASE_HOME/bin/hbase-daemon.sh stop master   
4. $HBASE_HOME/bin/hbase stop regionserver/zookeeper 不能这么调,调了也会出错,也没有路径会调用这个命令,但是可以通过$HBASE_HOME/bin/hbase start regionserver/zookeeper 来启动rs或者zk,hbase-daemon.sh调用的就是这个命令  

----------

## HTable一些基本概念
### Row key
> + 
行主键, HBase不支持条件查询和Order by等查询,读取记录只能按Row key(及其range)或全表扫描,因此Row key需要根据业务来设计以利用其存储排序特性(Table按Row key字典序排序如1,10,100,11,2)提高性能.
> + 
避免使用递增的数字或时间做为rowkey.
> + 
如果rowkey是整型,用二进制的方式比用string来存储更节约空间
> + 
合理的控制rowkey的长度,尽可能短,因为rowkey的数据也会存在每个Cell中.
> + 
如果需要将表预分裂为多个region是,最好自定义分裂的规则.

### Column Family(列族)
> + 
 在表创建时声明,每个Column Family为一个存储单元.在上例中设计了一个HBase表blog,该表有两个列族:article和author.    
> +  
 列簇尽量少,最好不超过3个.因为每个列簇是存在一个独立的HFile里的,flush和compaction操作都是针对一个Region进行的,当一个列簇的数据很多需要flush的时候,其它列簇即使数据很少也需要flush,这样就产生的大量不必要的io操作.
> +  
 在多列簇的情况下,注意各列簇数据的数量级要一致.如果两个列簇的数量级相差太大,会使数量级少的列簇的数据扫描效率低下.
> +  
 将经常查询和不经常查询的数据放到不同的列簇.
> +  
 因为列簇和列的名字会存在HBase的每个Cell中,所以他们的名字应该尽可能的短.比如,用f:q代替mycolumnfamily:mycolumnqualifier

### Column(列)
> +  
HBase的每个列都属于一个列族,以列族名为前缀,如列article:title和article:content属于article列族,author:name和author:nickname属于author列族.
> + 
Column不用创建表时定义即可以动态新增,同一Column Family的Columns会群聚在一个存储单元上,并依Column key排序,因此设计时应将具有相同I/O特性的Column设计在一个Column Family上以提高性能.同时这里需要注意的是:这个列是可以增加和删除的,这和我们的传统数据库很大的区别.所以他适合非结构化数据.

### Timestamp
> + 
HBase通过row和column确定一份数据,这份数据的值可能有多个版本,不同版本的值按照时间倒序排序,即最新的数据排在最前面,查询时默认返回最新版本.如上例中row key=1的author:nickname值有两个版本,分别为1317180070811对应的"一叶渡江"和1317180718830对应的"yedu"(对应到实际业务可以理解为在某时刻修改了nickname为yedu,但旧值仍然存在).Timestamp默认为系统当前时间(精确到毫秒),也可以在写入数据时指定该值.

### Value
> + 
每个值通过4个键唯一索引,tableName+RowKey+ColumnKey+Timestamp=>value,例如上例中{tableName='blog',RowKey='1',ColumnName='author:nickname',Timestamp=' 1317180718830'}索引到的唯一值是"yedu".

### 存储类型
>  
`TableName` 是字符串  
`RowKey`和`ColumnName`是二进制值(Java 类型 byte[])  
`Timestamp`是一个 64 位整数(Java 类型 long) 
`value`是一个字节数组(Java类型 byte[]).    

---    
## HBase配置优化
### zookeeper.session.timeout
> 
**默认值**:3分钟(180000ms)
> 
**说明**:RegionServer与Zookeeper间的连接超时时间.当超时时间到后,ReigonServer会被Zookeeper从RS集群清单中移除,HMaster收到移除通知后,会对这台server负责的regions重新balance,让其他存活的RegionServer接管.
> 
**调优**:这个timeout决定了RegionServer是否能够及时的failover.设置成1分钟或更低,可以减少因等待超时而被延长的failover时间.
不过需要注意的是,对于一些Online应用,RegionServer从宕机到恢复时间本身就很短的(网络闪断,crash等故障,运维可快速介入),如果调低timeout时间,反而会得不偿失.因为当ReigonServer被正式从RS集群中移除时,HMaster就开始做balance了(让其他RS根据故障机器记录的WAL日志进行恢复).当故障的RS在人工介入恢复后,这个balance动作是毫无意义的,反而会使负载不均匀,给RS带来更多负担.特别是那些固定分配regions的场景.

### hbase.regionserver.handler.count
> 
**默认值**:10
> 
**说明**:RegionServer的请求处理IO线程数.
> 
**调优**:
这个参数的调优与内存息息相关.
较少的IO线程,适用于处理单次请求内存消耗较高的Big PUT场景(大容量单次PUT或设置了较大cache的scan,均属于Big PUT)或ReigonServer的内存比较紧张的场景.
较多的IO线程,适用于单次请求内存消耗低,TPS要求非常高的场景.设置该值的时候,以监控内存为主要参考.
这里需要注意的是如果server的region数量很少,大量的请求都落在一个region上,因快速充满memstore触发flush导致的读写锁会影响全局TPS,不是IO线程数越高越好.
压测时,开启[Enabling RPC-level logging](http://hbase.apache.org/book.html#rpc.logging),  
可以同时监控每次请求的内存消耗和GC的状况,最后通过多次压测结果来合理调节IO线程数.
这里是一个案例?[Hadoop and HBase Optimization for Read Intensive Search Applications](http://software.intel.com/en-us/articles/hadoop-and-hbase-optimization-for-read-intensive-search-applications/),作者在SSD的机器上设置IO线程数为100,仅供参考.

### hbase.hregion.max.filesize
> 
**默认值**:256M
> 
**说明**:在当前ReigonServer上单个Reigon的最大存储空间,单个Region超过该值时,这个Region会被自动split成更小的region.
> 
**调优**:
小region对split和compaction友好,因为拆分region或compact小region里的storefile速度很快,内存占用低.缺点是split和compaction会很频繁.特别是数量较多的小region不停地split, compaction,会导致集群响应时间波动很大,region数量太多不仅给管理上带来麻烦,甚至会引发一些Hbase的bug.一般512以下的都算小region.  
大region,则不太适合经常split和compaction,因为做一次compact和split会产生较长时间的停顿,对应用的读写性能冲击非常大.此外,大region意味着较大的storefile,compaction时对内存也是一个挑战.
当然,大region也有其用武之地.如果你的应用场景中,某个时间点的访问量较低,那么在此时做compact和split,既能顺利完成split和compaction,又能保证绝大多数时间平稳的读写性能.  
既然split和compaction如此影响性能,有没有办法去掉?
compaction是无法避免的,split倒是可以从自动调整为手动.
只要通过将这个参数值调大到某个很难达到的值,比如100G,就可以间接禁用自动split(RegionServer不会对未到达100G的region做split).
再配合RegionSplitter这个工具,在需要split时,手动split.
手动split在灵活性和稳定性上比起自动split要高很多,相反,管理成本增加不多,比较推荐online实时系统使用.
内存方面,小region在设置memstore的大小值上比较灵活,大region则过大过小都不行,过大会导致flush时app的IO wait增高,过小则因store file过多影响读性能.

### hbase.regionserver.global.memstore.upperLimit/lowerLimit
> 
默认值:0.4/0.35
> 
**upperlimit说明**:hbase.hregion.memstore.flush.size 这个参数的作用是当单个Region内所有的memstore大小总和超过指定值时,flush该region的所有memstore.RegionServer的flush是通过将请求添加一个队列,模拟生产消费模式来异步处理的.那这里就有一个问题,当队列来不及消费,产生大量积压请求时,可能会导致内存陡增,最坏的情况是触发OOM.
这个参数的作用是防止内存占用过大,当ReigonServer内所有region的memstores所占用内存总和达到heap的40%时,HBase会强制block所有的更新并flush这些region以释放所有memstore占用的内存.
>  
**lowerLimit说明**: 同upperLimit,只不过lowerLimit在所有region的memstores所占用内存达到Heap的35%时,不flush所有的memstore.它会找一个memstore内存占用最大的region,做个别flush,此时写更新还是会被block.lowerLimit算是一个在所有region强制flush导致性能降低前的补救措施.在日志中,表现为 "** Flush thread woke up with memory above low water."  
>  
**调优**:这是一个Heap内存保护参数,默认值已经能适用大多数场景.
参数调整会影响读写,如果写的压力大导致经常超过这个阀值,则调小读缓存hfile.block.cache.size增大该阀值,或者Heap余量较多时,不修改读缓存大小.
如果在高压情况下,也没超过这个阀值,那么建议你适当调小这个阀值再做压测,确保触发次数不要太多,然后还有较多Heap余量的时候,调大hfile.block.cache.size提高读性能.
还有一种可能性是?hbase.hregion.memstore.flush.size保持不变,但RS维护了过多的region,要知道 region数量直接影响占用内存的大小.

### hfile.block.cache.size
> 
**默认值**:0.2
> 
**说明**:storefile的读缓存占用Heap的大小百分比,0.2表示20%.该值直接影响数据读的性能.
> 
**调优**:当然是越大越好,如果写比读少很多,开到0.4-0.5也没问题.如果读写较均衡,0.3左右.如果写比读多,果断默认吧.设置这个值的时候,你同时要参考?hbase.regionserver.global.memstore.upperLimit?,该值是memstore占heap的最大百分比,两个参数一个影响读,一个影响写.如果两值加起来超过80-90%,会有OOM的风险,谨慎设置.

### hbase.hstore.blockingStoreFiles
> 
**默认值**:7
> 
**说明**:在flush时,当一个region中的Store(Coulmn Family)内有超过7个storefile时,则block所有的写请求进行compaction,以减少storefile数量.
> 
**调优**:block写请求会严重影响当前regionServer的响应时间,但过多的storefile也会影响读性能.从实际应用来看,为了获取较平滑的响应时间,可将值设为无限大.如果能容忍响应时间出现较大的波峰波谷,那么默认或根据自身场景调整即可.

### hbase.hregion.memstore.block.multiplier
> 
**默认值**:2
> 
**说明**:当一个region里的memstore占用内存大小超过hbase.hregion.memstore.flush.size两倍的大小时,block该region的所有请求,进行flush,释放内存.
虽然我们设置了region所占用的memstores总内存大小,比如64M,但想象一下,在最后63.9M的时候,我Put了一个200M的数据,此时memstore的大小会瞬间暴涨到超过预期的hbase.hregion.memstore.flush.size的几倍.这个参数的作用是当memstore的大小增至超过hbase.hregion.memstore.flush.size 2倍时,block所有请求,遏制风险进一步扩大.
> 
**调优**: 这个参数的默认值还是比较靠谱的.如果你预估你的正常应用场景(不包括异常)不会出现突发写或写的量可控,那么保持默认值即可.如果正常情况下,你的写请求量就会经常暴长到正常的几倍,那么你应该调大这个倍数并调整其他参数值,比如hfile.block.cache.size和hbase.regionserver.global.memstore.upperLimit/lowerLimit,以预留更多内存,防止HBase server OOM.

### hbase.hregion.memstore.mslab.enabled
> 
**默认值**:true
> 
**说明**:减少因内存碎片导致的Full GC,提高整体性能.
> 
**调优**:详见 [http://kenwublog.com/avoid-full-gc-in-hbase-using-arena-allocation](http://kenwublog.com/avoid-full-gc-in-hbase-using-arena-allocation)


---  


