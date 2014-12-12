### (WJW)tomcat7安装文档

### [X] Linux环境: CentOs 6.2 X64 以上
* 彻底关闭SELINUX
* 关闭IPV6
* 防火墙开启8088端口
* 修改`/etc/profile`, 添加:
```
export JAVA_HOME=/usr/java/default
export PATH=$JAVA_HOME/bin:$PATH
```

* 修改`/etc/rc.d/rc.local`, 添加:
```
export JAVA_HOME=/usr/java/default
export PATH=$JAVA_HOME/bin:$PATH
```

* 修改`/etc/sysctl.conf`文件追加
```
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
kernel.pid_max = 65536
fs.aio-max-nr = 1048576
fs.file-max = 6815744
>
net.core.rmem_default = 8388608
net.core.rmem_max = 8388608
net.core.wmem_default = 8388608
net.core.wmem_max = 8388608
>
net.core.somaxconn = 262144
net.ipv4.tcp_no_metrics_save = 1
net.core.netdev_max_backlog = 262144
>
net.ipv4.tcp_rmem = 8192 4194304 8388608
net.ipv4.tcp_wmem = 4096 2097152 8388608
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_timestsmps = 0
net.ipv4.tcp_retries2 = 5
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_max_orphans = 262144
net.ipv4.conf.lo.arp_ignore = 0
net.ipv4.conf.lo.arp_announce = 0
net.ipv4.conf.all.arp_ignore = 0
net.ipv4.conf.all.arp_announce = 0
net.nf_conntrack_max = 655360
net.netfilter.nf_conntrack_tcp_timeout_established = 180
```

执行#/sbin/sysctl -p 命令操作来使我们所做的变更生效.

* `/etc/security/limits.conf`文件追加
```
*               soft    nproc   16384`
*               hard    nproc   16384`
*               soft    nofile  65536`
*               hard    nofile  65536`
```

* `/etc/pam.d/login`文件追加
```
session    required     /lib64/security/pam_limits.so
```

### [X] JDK安装
* Sun的JDK 1.7 Update 40以上
* 需要rpm安装

### [X] Tomcat安装
* 创建www用户
```
groupadd www -g 58
useradd -u 58 -g www www
```

* 创建/data/app目录:`mkdir -p /data/app`
* 复制tomcat7.tar.gz到/data/app目录下,解压:`tar zxvf tomcat7.tar.gz`
* 改变/data/app/tomcat7的属主:`chown -R www:www /data/app/tomcat7`
* 添加可执行权限:
```
chmod +x /data/app/tomcat7/bin/tomcat
chmod +x /data/app/tomcat7/wrapper_home/sbin/wrapper-linux-x86-64
```

### [X] Tomcat配置
* Java虚拟机的配置文件在: `/data/app/tomcat7/wrapper_home/conf/wrapper.conf`文件里,可修改的参数是:
```
wrapper.java.additional.16=-Xms2g
wrapper.java.additional.17=-Xmx8g
wrapper.java.additional.18=-XX:MaxPermSize=512m
wrapper.java.additional.19=-XX:+UseConcMarkSweepGC
wrapper.java.additional.20=-XX:+CMSClassUnloadingEnabled
wrapper.java.additional.21=-XX:+DoEscapeAnalysis
wrapper.java.additional.22=-Dspring.profiles.active=prod
```

* 修改Tomcat参数的的配置文件在: `/data/app/tomcat7/conf/server.xml`文件里,可修改的参数是:
```
     <Executor name="tomcatThreadPool" namePrefix="tomcatThreadPool-"
                  maxThreads="1000" minSpareThreads="20" maxIdleTime="60000" />


     <Connector executor="tomcatThreadPool"
                port="8088" protocol="HTTP/1.1"
                connectionTimeout="300000"
                keepAliveTimeout="15000"
                maxKeepAliveRequests="100"
                redirectPort="8443"
                maxHttpHeaderSize="8192" URIEncoding="UTF-8" enableLookups="false" acceptCount="200" disableUploadTimeout="true"/>
```

* tomcat的状态日志: `/data/app/tomcat7/logs/wrapper-tomcat7.log`
* tomcat存取日志: `/data/app/tomcat7/logs/localhost_access_log.YYYY-MM-DD.txt`

### Tomcat的调试,启动,停止:
>   
Usage: /data/app/tomcat7/bin/tomcat [ console | start | stop | restart | status ]
 Commands:
   `console`      Launch in the current console.
   `start`        Start in the background as a daemon process.
   `stop`         Stop if running as a daemon or in another console.
   `restart`      Stop if running and then start.
   `status`       Query the current status.
>

### [X] 附录:
#####  [X] 开机启动tomcat
  修改/etc/rc.d/rc.local, 添加:```su www -lc '/data/app/tomcat7/bin/tomcat start'```

##### [X] 安装APR
1.如果没有安装SSL,就先安装SSL,首选用:`yum -y install openssl openssl-devel`
或者,用源代码安装:解压缩openssl-1.0.1e.tar.gz
```
tar -zxvf openssl-1.0.1e.tar.gz
cd ./openssl-1.0.1e
./configure -fPIC enable-shared
make
make install
```

APR会生成在:`/usr/local/ssl/`目录下

2.解压缩apr-1.4.8.tar.gz
```
tar -zxvf apr-1.4.8.tar.gz
cd ./apr-1.4.8
./configure
make
make install
```

APR会生成在:`/usr/local/apr/`目录下

3.安装apr-util-1.5.2.tar.gz
```
tar -zxvf apr-util-1.5.2.tar.gz
cd ./apr-util-1.5.2
./configure --with-apr=/usr/local/apr
make
make install
```

4.解压缩tomcat-native-1.1.29-src.tar.gz
```
tar -zxvf ./tomcat-native-1.1.29-src.tar.gz
cd ./tomcat-native-1.1.29-src/jni/native/
./configure --with-apr=/usr/local/apr --with-ssl=yes
make
make install
```
