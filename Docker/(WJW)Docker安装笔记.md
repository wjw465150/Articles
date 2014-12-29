### (WJW)Docker安装笔记

#### [X] 一、禁用selinux
由于Selinux和LXC有冲突,所以需要禁用selinux.编辑/etc/selinux/config,设置两个关键变量.
```
SELINUX=disabled
SELINUXTYPE=targeted
```

#### [X] 二、安装Fedora EPEL源
```bash
rpm -ivh ./epel-release-6-8.noarch.rpm  
或者:  
#yum install http://ftp.riken.jp/Linux/fedora/epel/6/x86_64/epel-release-6-8.noarch.rpm
```

#### [X] 三、添加hop5.repo源
```bash
cp ./hop5.repo /etc/yum.repos.d        
或者:  
#cd /etc/yum.repos.d        
#wget http://www.hop5.in/yum/el6/hop5.repo  
```

#### [X] 四、安装Docker
#### yum安装
##### [X] 升级带aufs模块的3.10内核(可选)
```bash
yum install kernel-ml-aufs kernel-ml-aufs-devel
```
> 1. 修改grub的主配置文件/etc/grub.conf，设置default=0，表示第一个title下的内容为默认启动的kernel（一般新安装的内核在第一个位置）。
> 2. 重启系统`reboot now`,
> 3. 然后执行`uname -r`,查看是否已经是3.10内核!
>>   
>> [root@localhost ~]# uname -r  
>> 3.10.5-3.el6.x86_64  
>>   
  
> 4. 执行`grep aufs /proc/filesystems`,查看内核是否支持aufs!
>>   
>>  [root@localhost ~]# grep aufs /proc/filesystems  
>>  nodev    aufs
>>   

##### [X] 安装docker
```bash
yum install redhat-lsb

yum install iptables
chkconfig ip6tables off
chkconfig iptables off
service ip6tables stop
service iptables stop

yum install device-mapper-libs
yum install libcgroup*
yum install docker-io
```
如果报错:`Error: Cannot retrieve metalink for repository: epel. Please verify its path and try again`  
```
解决办法是编辑/etc/yum.repos.d/epel.repo，把基础的恢复，镜像的地址注释掉

#baseurl
mirrorlist

改成

baseurl
#mirrorlist

然后执行: yum clean all
```

```
=============================================================================================================================================================================================================================================
 Package                                                       Arch                                                  Version                                                       Repository                                           Size
=============================================================================================================================================================================================================================================
Installing:
 docker-io                                                     x86_64                                                1.2.0-3.el6                                                   epel                                                4.2 M
Installing for dependencies:
 lua-alt-getopt                                                noarch                                                0.7.0-1.el6                                                   epel                                                6.9 k
 lua-filesystem                                                x86_64                                                1.4.2-1.el6                                                   epel                                                 24 k
 lua-lxc                                                       x86_64                                                1.0.6-1.el6                                                   epel                                                 15 k
 lxc                                                           x86_64                                                1.0.6-1.el6                                                   epel                                                120 k
 lxc-libs                                                      x86_64                                                1.0.6-1.el6                                                   epel                                                248 k

Transaction Summary
=============================================================================================================================================================================================================================================
Install       6 Package(s)

Downloading Packages:
(1/6): docker-io-1.2.0-3.el6.x86_64.rpm                                                                                                                                                                               | 4.2 MB     00:02
(2/6): lua-alt-getopt-0.7.0-1.el6.noarch.rpm                                                                                                                                                                          | 6.9 kB     00:00
(3/6): lua-filesystem-1.4.2-1.el6.x86_64.rpm                                                                                                                                                                          |  24 kB     00:00
(4/6): lua-lxc-1.0.6-1.el6.x86_64.rpm                                                                                                                                                                                 |  15 kB     00:00
(5/6): lxc-1.0.6-1.el6.x86_64.rpm                                                                                                                                                                                     | 120 kB     00:00
(6/6): lxc-libs-1.0.6-1.el6.x86_64.rpm                                                                                                                                                                                | 248 kB     00:00
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

#### rpm手工安装:  
先执行
```bash
rpm --nodeps -ev $(rpm -qa | grep -i 'device-mapper-libs-')
rpm --nodeps -ev $(rpm -qa | grep -i 'device-mapper-')
rpm --nodeps -ev $(rpm -qa | grep -i 'device-mapper-event-libs-')
rpm --nodeps -ev $(rpm -qa | grep -i 'device-mapper-event-')

rpm -ivh ./device-mapper-event-1.02.90-2.el6.x86_64.rpm ./device-mapper-event-libs-1.02.90-2.el6.x86_64.rpm ./device-mapper-1.02.90-2.el6.x86_64.rpm ./device-mapper-libs-1.02.90-2.el6.x86_64.rpm
```

再执行
```bash
rpm -ivh ./lxc-libs-1.0.6-1.el6.x86_64.rpm
rpm -ivh ./lua-alt-getopt-0.7.0-1.el6.noarch.rpm
rpm -ivh ./lua-filesystem-1.4.2-1.el6.x86_64.rpm
rpm -ivh ./lua-lxc-1.0.6-1.el6.x86_64.rpm
rpm -ivh ./lxc-1.0.6-1.el6.x86_64.rpm
#6.4版本以下需要 rpm -ivh ./libcgroup-0.40.rc1-5.el6.x86_64.rpm
rpm -ivh ./docker-io-1.3.2-2.el6.x86_64.rpm
```
#### [X] 五、初步验证docker
  输入`docker -h`,如果有如下输出,就证明docker在形式上已经安装成功.
```
# docker -h 
Usage of docker: 
  -D=false: Enable debug mode 
  -H=[]: Multiple tcp://host:port or unix://path/to/socket to bind in daemon mode, single connection otherwise 
  -api-enable-cors=false: Enable CORS headers in the remote API 
  -b="": Attach containers to a pre-existing network bridge; use 'none' to disable container networking
  -bip="": Use this CIDR notation address for the network bridge's IP, not compatible with -b 
  -d=false: Enable daemon mode 
  -dns=[]: Force docker to use specific DNS servers 
  -g="/var/lib/docker": Path to use as the root of the docker runtime 
  -icc=true: Enable inter-container communication 
  -ip="0.0.0.0": Default IP address to use when binding container ports 
  -iptables=true: Disable docker's addition of iptables rules 
  -p="/var/run/docker.pid": Path to use for daemon PID file
  -r=true: Restart previously running containers 
  -s="": Force the docker runtime to use a specific storage driver 
  -v=false: Print version information and quit
```

#### [X] 六 、启动docker服务:
```
service docker start
#docker -d -H unix:///var/run/docker.sock
```
#### [X] 七 、查找映象:
`docker search 映象名字`
从index.docker.io搜寻所需镜像.
去https://index.docker.io获取镜像相关的信息.

#### [X] 八 、进入 Docker 容器:
创建`vi /bin/docker-enter.sh`
```bash
#!/bin/sh

if [ -e $(dirname "$0")/nsenter ]; then
  # with boot2docker, nsenter is not in the PATH but it is in the same folder
  NSENTER=$(dirname "$0")/nsenter
else
  NSENTER=nsenter
fi

if [ -z "$1" ]; then
  echo "Usage: `basename "$0"` CONTAINER [COMMAND [ARG]...]"
  echo ""
  echo "Enters the Docker CONTAINER and executes the specified COMMAND."
  echo "If COMMAND is not specified, runs an interactive shell in CONTAINER."
else
  PID=$(docker inspect --format "{{.State.Pid}}" "$1")
  if [ -z "$PID" ]; then
    exit 1
  fi
  shift

  OPTS="--target $PID --mount --uts --ipc --net --pid --"

  if [ -z "$1" ]; then
    # No command given.
    # Use su to clear all host environment variables except for TERM,
    # initialize the environment variables HOME, SHELL, USER, LOGNAME, PATH,
    # and start a login shell.
    "$NSENTER" $OPTS su - root
  else
    # Use env to clear all host environment variables.
    "$NSENTER" $OPTS env --ignore-environment -- "$@"
  fi
fi
```
加上可执行`chmod +x /bin/docker-enter.sh`  
运行`docker-enter.sh <container id>`,这样就进入到指定的容器中!  

#### [X] 附录:低版本的Redhat(6.3)可能要手动挂载cgroup
  我们首选禁用cgroup对应服务cgconfig.
```bash
  service cgconfig stop # 关闭服务 
  chkconfig cgconfig off # 取消开机启动
```
  然后挂载cgroup,可以命令行挂载
```bash
  mount -t cgroup none /cgroup  #仅本次有效
```
  或者修改配置文件,编辑`/etc/fstab`,加入
```
none                    /cgroup                 cgroup  defaults        0 0
```

