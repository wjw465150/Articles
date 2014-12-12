## (WJW)Memcached高可用安装笔记

### [x] 安装环境介绍:
```
Master: T1
Slave: T2
VIP: 192.168.68.46
```
```
memcached-1.2.8-repcached-2.2.1,是memcached1.2.x的扩展，目的是实现memcached的数据复制属性，来实现memcached的数据备份。该扩展可以使memcached通过异步数据复制实现memcached的数据复制/备份功能，同时支持Memcached的所有操作指令。
它是一个单master单slave的方案,但它的 master/slave都是可读写的,而且可以相互同步,依赖libevent-1.4.X库,不能使用libevent-2.X库
但是要注意安装memcached后, 如果再安装repcached的完整安装包,会有冲突.
如果已经安装了memcached,打算换成repcached,卸载memcached,然后安装完整repcached包
如果安装的系统是64位的,可以增加选项: --enable-64bit
注意: memcached-repcached只支持文本协议!
```

### [x] 安装Memcached的repcached版本(Master,Slave)
#### 1.安装libevent(http://www.monkey.org/~provos/libevent-1.4.13-stable.tar.gz)
```
tar -zxvf libevent-1.4.13-stable.tar.gz
cd libevent-1.4.13-stable
./configure --prefix=/usr/local
make uninstall
make clean
make
make install
cd ..
```

#### 2.安装memcached-repcached(http://downloads.sourceforge.net/repcached/memcached-1.2.8-repcached-2.2.1.tar.gz)
```
tar -zxvf memcached-1.2.8-repcached-2.2.1.tar.gz
cd memcached-1.2.8-repcached-2.2.1
./configure --with-libevent=/usr/local/lib/ --enable-replication --enable-64bit
make uninstall
make clean
make
make install
```
**注意:如果make的时候报错**
```
memcached.c: 在函数'add_iov'中:
memcached.c:696:30: 错误: 'IOV_MAX'未声明(在此函数内第一次使用)
memcached.c:696:30: 附注: 每个未声明的标识符在其出现的函数内只报告一次
make[2]: *** [memcached-memcached.o] 错误 1
需要修改 memcached.c 文件:

/* FreeBSD 4.x doesn't have IOV_MAX exposed. */
#ifndef IOV_MAX
#if defined(__FreeBSD__) || defined(__APPLE__)
# define IOV_MAX 1024
#endif
#endif

改成:
/* FreeBSD 4.x doesn't have IOV_MAX exposed. */
#ifndef IOV_MAX
# define IOV_MAX 1024
#endif
```
**注意:如果运行`/usr/local/bin/memcached -h`的时候报错:`找不到libevent`**  
执行: `ln -s /usr/local/lib/libevent-1.4.so.2 /usr/lib64/libevent-1.4.so.2`  
技巧: `执行ldd /usr/local/bin/memcached`,看缺少那些库!  

#### 3. Master上的Memcached启动停止脚本:/etc/rc.d/init.d/memcached(注意:文件格式一定要是unix的)
编辑: `vi /etc/rc.d/init.d/memcached`
```bash
#!/bin/sh
#
# memcached: MemCached Daemon
#
# chkconfig: - 90 25
# description: MemCached Daemon
#
# Source function library.
. /etc/rc.d/init.d/functions
. /etc/sysconfig/network
#[ ${NETWORKING} = "no" ] && exit 0
#[ -r /etc/sysconfig/dund ] || exit 0
#. /etc/sysconfig/dund
#[ -z "$DUNDARGS" ] && exit 0

start() {
  echo -n $"Starting memcached: "

  daemon $MEMCACHED -d -u daemon -P ${PID_FILE} -m 200 -l 0.0.0.0 -p 11311 -X 11411 -x T2

  echo
}

stop() {
  echo -n $"Shutting down memcached: "
  killproc -p ${PID_FILE} memcached
  echo
}

MEMCACHED="/usr/local/bin/memcached"
PID_FILE="/var/run/memcached_repcached.pid"

[ -f $MEMCACHED ] || exit 1

# See how we were called.
case "$1" in
  start)
    start
    ;;
  stop)
    stop
    ;;
  restart)
    stop
    sleep 3
    start
    ;;
  status)
    status -p ${PID_FILE} memcached
    ;;
  *)
    echo $"Usage: $0 {start|stop|restart|status}"
    exit 1
esac

exit 0
```

变成service服务脚本: 
```
chmod +x /etc/rc.d/init.d/memcached
chkconfig --add memcached
#chkconfig --level 35 memcached on
```

#### 4. Slave上的Memcached启动停止脚本:/etc/rc.d/init.d/memcached(注意:文件格式一定要是unix的)
编辑: `vi /etc/rc.d/init.d/memcached`
```bash
#!/bin/sh
#
# memcached: MemCached Daemon
#
# chkconfig: - 90 25
# description: MemCached Daemon
#
# Source function library.
. /etc/rc.d/init.d/functions
. /etc/sysconfig/network
#[ ${NETWORKING} = "no" ] && exit 0
#[ -r /etc/sysconfig/dund ] || exit 0
#. /etc/sysconfig/dund
#[ -z "$DUNDARGS" ] && exit 0

start() {
  echo -n $"Starting memcached: "

  daemon $MEMCACHED -d -u daemon -P ${PID_FILE} -m 200 -l 0.0.0.0 -p 11311 -X 11411 -x T1

  echo
}

stop() {
  echo -n $"Shutting down memcached: "
  killproc -p ${PID_FILE} memcached
  echo
}

MEMCACHED="/usr/local/bin/memcached"
PID_FILE="/var/run/memcached_repcached.pid"

[ -f $MEMCACHED ] || exit 1

# See how we were called.
case "$1" in
  start)
    start
    ;;
  stop)
    stop
    ;;
  restart)
    stop
    sleep 3
    start
    ;;
  status)
    status -p ${PID_FILE} memcached
    ;;
  *)
    echo $"Usage: $0 {start|stop|restart|status}"
    exit 1
esac

exit 0
```

变成service服务脚本: 
```
chmod +x /etc/rc.d/init.d/memcached
chkconfig --add memcached
#chkconfig --level 35 memcached on
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
   router_id LVS_MEMCACHED
}

vrrp_script chk_memcached {
  script "/etc/keepalived/scripts/memcached_check.sh"   ###监控脚本
  interval 2                                        ###监控时间
}

vrrp_instance VI_1 {
    nopreempt ###不抢占,防止脑裂和同步数据时get不命中的问题!
    state MASTER             #备的是BACKUP
    interface br0
    virtual_router_id 51
    priority 100            #备的是90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass memcached
    }
    virtual_ipaddress {
        192.168.68.46
    }
    
    track_script {
      chk_memcached                       ###执行上面定义的chk_memcached
    }
    
    notify_master /etc/keepalived/scripts/memcached_master.sh
    notify_backup /etc/keepalived/scripts/memcached_backup.sh
    notify_fault  /etc/keepalived/scripts/memcached_fault.sh
    notify_stop   /etc/keepalived/scripts/memcached_stop.sh
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
   router_id LVS_MEMCACHED
}

vrrp_script chk_memcached {
  script "/etc/keepalived/scripts/memcached_check.sh"   ###监控脚本
  interval 2                                        ###监控时间
}

vrrp_instance VI_1 {
    nopreempt ###不抢占,防止脑裂和同步数据时get不命中的问题!
    state BACKUP             #备的是BACKUP
    interface br0
    virtual_router_id 51
    priority 90            #备的是90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass memcached
    }
    virtual_ipaddress {
        192.168.68.46
    }
    
    track_script {
      chk_memcached                       ###执行上面定义的chk_memcached
    }
    
    notify_master /etc/keepalived/scripts/memcached_master.sh
    notify_backup /etc/keepalived/scripts/memcached_backup.sh
    notify_fault  /etc/keepalived/scripts/memcached_fault.sh
    notify_stop   /etc/keepalived/scripts/memcached_stop.sh
}
```

#### [x] 在Master和Slave上创建监控memcached的脚本:
##### chk_memcached
`mkdir /etc/keepalived/scripts/`
`vi /etc/keepalived/scripts/memcached_check.sh`

```bash
#!/bin/bash

C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`

kill -0 `cat /var/run/memcached_repcached.pid`

if [ "$?" -eq 0 ]; then
 echo "${C_DATE} memcached is ALIVE"
 exit 0
else
 echo "${C_DATE} memcached is DOWN"
 exit 1
fi
```

##### notity_master
`vim /etc/keepalived/scripts/memcached_master.sh`
```bash
#!/bin/bash
 
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
LOGFILE="/var/log/keepalived-memcached-state.log"
 
echo "${C_DATE} [master]" >> $LOGFILE
```

##### notify_backup
`vim /etc/keepalived/scripts/memcached_backup.sh`
```bash
#!/bin/bash
 
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
LOGFILE="/var/log/keepalived-memcached-state.log"
 
echo "${C_DATE} [backup]" >> $LOGFILE
```

##### notify_fault
`vim /etc/keepalived/scripts/memcached_fault.sh`
```bash
#!/bin/bash
 
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
LOGFILE=/var/log/keepalived-memcached-state.log
 
echo "${C_DATE} [fault]" >> $LOGFILE
```

##### notify_stop
`vim /etc/keepalived/scripts/memcached_stop.sh`
```bash
#!/bin/bash
 
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`
LOGFILE=/var/log/keepalived-memcached-state.log
 
echo "${C_DATE} [stop]" >> $LOGFILE
```

#### [x] 给监控脚本都加上可执行权限：
`chmod +x /etc/keepalived/scripts/*.sh`

### [x] 启动步骤:
1. 启动Master上的Memcached
`service memcached start`

2. 启动Slave上的Memcached
`service memcached start`

3. 启动Master上的Keepalived
`service keepalived start`

4. 启动Slave上的Keepalived
`service keepalived start`

### [X] 附录:
#### 1.手工启动memcache
```
#这里的-u memcache 必须与启动memcached 进程的用户保持一直，否则无法完成复制！
主节点1: (11311是服务监听端口,11411是数据同步端口;-x 指定replicaIP,-X指定replica监听端口) 
/usr/local/bin/memcached -m 200 -v -u root -l 0.0.0.0 -p 11311 -X 11411 -x ${主节点2}

主节点2: (11311是服务监听端口,11411是数据同步端口;-x 指定replicaIP,-X指定replica监听端口)
/usr/local/bin/memcached -m 200 -v -u root -l 0.0.0.0 -p 11311 -X 11411 -x ${主节点1} 
```
