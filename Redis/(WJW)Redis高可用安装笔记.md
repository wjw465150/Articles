## (WJW)Redis高可用安装笔记

### [x] 安装环境介绍:
```
Master: T1
Slave: T2
VIP: 192.168.68.45
```

### [x] 安装Redis(Master,Slave)
```
vi /etc/sysctl.conf
添加一行: `vm.overcommit_memory=1`

mkdir /opt/redis
mkdir /opt/redis/log
mkdir /opt/redis/db

tar zxvf ./redis-2.8.17.tar.gz
cd redis-2.8.17
make PREFIX=/opt/redis install
```

#### [x] Redis启动脚本(Master,Slave):/opt/redis/bin/startRedis.sh
```
#!/bin/bash

basedir=`dirname $0`
echo "Redis BASE DIR:$basedir"
cd $basedir

nohup ./redis-server ./redis.conf > /dev/null 2>&1 &
```

#### [x] Redis停止脚本(Master,Slave):/opt/redis/bin/StopRedis.sh
```
#!/bin/sh

basedir=`dirname $0`
echo "Redis BASE DIR:$basedir"
cd $basedir

./redis-cli -h localhost -a 123456 shutdown
```

#### [x] Redis配置文件(Master,Slave):/opt/redis/bin/redis.conf
```
#requirepass 123456

pidfile /opt/redis/bin/redis.pid
logfile /opt/redis/log/redis.log
dir /opt/redis/db/

daemonize yes
port 6379
timeout 300
loglevel warning
databases 16
maxmemory 1g

#不要快照
#save 900 1
#save 300 10
#save 60 10000
#rdbcompression yes
#dbfilename dump.rdb

#使用AOF
appendonly yes
appendfsync everysec
```

### [x] 安装Keepalived(Master,Slave):
```
#wget http://www.keepalived.org/software/keepalived-1.2.13.tar.gz
tar zxvf keepalived-1.2.13.tar.gz
cd ./keepalived-1.2.13
./configure
make 
make install

cp /usr/local/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/
cp /usr/local/etc/sysconfig/keepalived /etc/sysconfig/
cp /usr/local/sbin/keepalived /usr/sbin/
mkdir /etc/keepalived
cp /usr/local/etc/keepalived/keepalived.conf /etc/keepalived/
chkconfig --add keepalived
#chkconfig --level 35 keepalived on
#service keepalived start
```

#### [x]  notify_*解释
```
Keepalived在转换状态时会依照状态来呼叫：
当进入Master状态时会呼叫notify_master
当进入Backup状态时会呼叫notify_backup
当发现异常情况时(track_script,track_interface失败)进入Fault状态呼叫notify_fault
当Keepalived程序终止时则呼叫notify_stop
```

#### [x] 在Master上创建如下配置文件：
`vim /etc/keepalived/keepalived.conf`
```
! Configuration File for keepalived

global_defs {
   notification_email {
     root@localhost
   }
   notification_email_from root@localhost
   smtp_server localhost
   smtp_connect_timeout 30
   router_id LVS_REDIS
}

vrrp_script chk_redis {
  script "/etc/keepalived/scripts/redis_check.sh"   ###监控脚本
  interval 2                                        ###监控时间
}

vrrp_instance VI_1 {
    nopreempt ###不抢占,防止脑裂
    state MASTER             #备的是BACKUP
    interface br0
    virtual_router_id 51
    priority 100            #备的是90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass redis
    }
    virtual_ipaddress {
        192.168.68.45
    }
    
    track_script {
      chk_redis                       ###执行上面定义的chk_redis
    }
    
    notify_master /etc/keepalived/scripts/redis_master.sh
    notify_backup /etc/keepalived/scripts/redis_backup.sh
    notify_fault  /etc/keepalived/scripts/redis_fault.sh
    notify_stop   /etc/keepalived/scripts/redis_stop.sh
}
```

#### [x] 在Slave上创建如下配置文件：
`vim /etc/keepalived/keepalived.conf`
```
! Configuration File for keepalived

global_defs {
   notification_email {
     root@localhost
   }
   notification_email_from root@localhost
   smtp_server localhost
   smtp_connect_timeout 30
   router_id LVS_REDIS
}

vrrp_script chk_redis {
  script "/etc/keepalived/scripts/redis_check.sh"   ###监控脚本
  interval 2                                        ###监控时间
}

vrrp_instance VI_1 {
    nopreempt ###不抢占,防止脑裂
    state BACKUP             #备的是BACKUP
    interface br0
    virtual_router_id 51
    priority 90            #备的是90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass redis
    }
    virtual_ipaddress {
        192.168.68.45
    }
    
    track_script {
      chk_redis                       ###执行上面定义的chk_redis
    }
    
    notify_master /etc/keepalived/scripts/redis_master.sh
    notify_backup /etc/keepalived/scripts/redis_backup.sh
    notify_fault  /etc/keepalived/scripts/redis_fault.sh
    notify_stop   /etc/keepalived/scripts/redis_stop.sh
}
```

#### [x] 在Master和Slave上创建监控Redis的脚本:
`mkdir /etc/keepalived/scripts/`
`vi /etc/keepalived/scripts/redis_check.sh`
```
#!/bin/bash

C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
ALIVE=`/opt/redis/bin/redis-cli PING`

if [ "$ALIVE" == "PONG" ]; then
  echo "${C_DATE} $ALIVE"
  exit 0
else
  echo "${C_DATE} $ALIVE"
  exit 1
fi
```

#### [x] 在Master与Slave创建如下脚本`notify_fault`和`notify_stop`：
`vim /etc/keepalived/scripts/redis_fault.sh`
```
#!/bin/bash
 
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
LOGFILE=/var/log/keepalived-redis-state.log
 
echo "${C_DATE} [fault]" >> $LOGFILE
```

`vim /etc/keepalived/scripts/redis_stop.sh`
```
#!/bin/bash
 
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
LOGFILE=/var/log/keepalived-redis-state.log
 
echo "${C_DATE} [stop]" >> $LOGFILE
```

#### [x] 在Master上创建`notity_master`与`notify_backup`脚本:
`vim /etc/keepalived/scripts/redis_master.sh`
```
#!/bin/bash
 
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
REDISCLI="/opt/redis/bin/redis-cli"
LOGFILE="/var/log/keepalived-redis-state.log"
 
echo "${C_DATE} [master]" >> $LOGFILE

#当keepalived配置为"抢占式"时,打开下面注释
#echo "Being master...." >> $LOGFILE 2>&1
#echo "Run SLAVEOF cmd ..." >> $LOGFILE
#$REDISCLI SLAVEOF T2 6379 >> $LOGFILE  2>&1
#sleep 10 #延迟10秒以后待数据同步完成后再取消同步状态
 
echo "Run SLAVEOF NO ONE cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF NO ONE >> $LOGFILE 2>&1
```

`vim /etc/keepalived/scripts/redis_backup.sh`
```
#!/bin/bash
 
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
REDISCLI="/opt/redis/bin/redis-cli"
LOGFILE="/var/log/keepalived-redis-state.log"
 
echo "${C_DATE} [backup]" >> $LOGFILE

#当keepalived配置为"抢占式"时,打开下面注释
#echo "Being slave...." >> $LOGFILE 2>&1
#sleep 15 #延迟15秒待数据被对方同步完成之后再切换主从角色

echo "Run SLAVEOF cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF T2 6379 >> $LOGFILE  2>&1
```

#### [x] 在Slave上创建`notity_master`与`notify_backup`脚本:
`vim /etc/keepalived/scripts/redis_master.sh`
```
#!/bin/bash
 
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
REDISCLI="/opt/redis/bin/redis-cli"
LOGFILE="/var/log/keepalived-redis-state.log"
 
echo "${C_DATE} [master]" >> $LOGFILE

#当keepalived配置为"抢占式"时,打开下面注释
#echo "Being master...." >> $LOGFILE 2>&1
#echo "Run SLAVEOF cmd ..." >> $LOGFILE
#$REDISCLI SLAVEOF T1 6379 >> $LOGFILE  2>&1
#sleep 10 #延迟10秒以后待数据同步完成后再取消同步状态
 
echo "Run SLAVEOF NO ONE cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF NO ONE >> $LOGFILE 2>&1
```

`vim /etc/keepalived/scripts/redis_backup.sh`
```
#!/bin/bash
 
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
REDISCLI="/opt/redis/bin/redis-cli"
LOGFILE="/var/log/keepalived-redis-state.log"
 
echo "${C_DATE} [backup]" >> $LOGFILE

#当keepalived配置为"抢占式"时,打开下面注释
#echo "Being slave...." >> $LOGFILE 2>&1
#sleep 15 #延迟15秒待数据被对方同步完成之后再切换主从角色

echo "Run SLAVEOF cmd ..." >> $LOGFILE
$REDISCLI SLAVEOF T1 6379 >> $LOGFILE  2>&1
```

#### [x] 在Master和Slave上,给监控脚本都加上可执行权限：
`chmod +x /etc/keepalived/scripts/*.sh`

### [x] 启动步骤:
1. 启动Master上的Redis
`/opt/redis/bin/startRedis.sh`

2. 启动Slave上的Redis
`/opt/redis/bin/startRedis.sh`

3. 启动Master上的Keepalived
`service keepalived start`

4. 启动Slave上的Keepalived
`service keepalived start`


