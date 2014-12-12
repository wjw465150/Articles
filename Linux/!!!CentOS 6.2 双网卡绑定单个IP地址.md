###加载bonding模块: `modprobe bonding`

###确认模块是否加载成功: `lsmod | grep bonding`

###停止NeetworkManager
`service NetworkManager stop`

`chkconfig NetworkManager off`

###在/etc/sysconfig/network-scripts/目录下建立ifcfg-bond0文件，文件内容如下：
```
DEVICE=bond0
BOOTPROTO=static
ONBOOT=yes
BONDING_OPTS="mode=0 miimon=100"
IPADDR=192.168.100.17
NETMASK=255.255.255.0
GATEWAY=192.168.100.1
```

###然后分别修改ifcfg-eth0文件，如下：
```
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
USERCTL=no
MASTER=bond0
SLAVE=yes
```
###在把ifcfg-eth1文件修改如下：
```
DEVICE=eth1
BOOTPROTO=none
ONBOOT=yes
USERCTL=no
MASTER=bond0
SLAVE=yes
```

###在/etc/modprobe.d/目录下建立modprobe.conf文件(`vi /etc/modprobe.d/modprobe.conf`)，文件内容如下：
```
alias netdev-bond0 bonding
```

###编辑 /etc/sysctl.conf
```
 net.bridge.bridge-nf-call-iptables=1  
 net.ipv4.ip_forward=1
```

###将bond0桥接到桥接器br0
>  brctl addbr br0 #创建一个桥接口  
>  brctl addif br0 bond0     #重要,add interface to bridge  
>  brctl show

###然后重启网络: service network restart
之后就可以用ifconfig -a看到绑定好的bond0网卡，bond0与eth0，eth1的mac地址均为一样。
可以同过cat /proc/net/bonding/bond0 此命令查看绑定情况

======
说明：
miimon是用来进行链路监测的。 比如：miimon=100，单位是ms(毫秒)这边的100，是100ms，即是0.1秒那么系统每100ms监测一次链路连接状态，如果有一条线路不通就转入另一条线路；	
mode的值表示工作模式，他共有0，1,2,3四种模式，常用的为0、1两种。	 
mode共有七种(0~6)，这里解释两个常用的选项。	
`mode=0`：表示load balancing (round-robin)为负载均衡方式，两块网卡都在工作。	
`mode=1`：表示fault-tolerance (active-backup)提供冗余功能，工作方式是主备的工作方式，其中一块网卡在工作（若eth0断掉），则自动切换到另一个块网卡（eth1做备份）。	
bonding只能提供链路监测，即从主机到交换机的链路是否接通。如果只是交换机对外的链路down掉了，而交换机本身并没有故障，那么bonding会认为链路没有问题而继续使用。	
