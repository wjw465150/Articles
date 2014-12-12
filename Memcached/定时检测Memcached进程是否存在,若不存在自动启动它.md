#### [X] 定时检测Memcached进程是否存在,若不存在自动启动它
----------
下面脚本会自动检测Memcached的进程,如果挂掉则自动重启Memcached服务.
`mkdir /opt/sh/`
`vi /opt/sh/memcached_check.sh`
```bash
#!/bin/sh

#check memcached process and restart if down
C_DATE=`date +"[%Y-%m-%d %H:%M:%S]"`

MC_CMD="/usr/local/bin/memcached"
MC_PORT=11211
MC_MEMORY=1024
LOG_FILE="/var/log/memcached_check.logs"

#用netstat命令查看memcached进程
MM=`netstat -lnp | grep "memcached" | grep "${MC_PORT}" | grep -v "grep" |wc -l`

#if语句判断进程是否存在,如果不存在,输出日志记录并重启memcached服务
if [ "$MM" -eq "0" ]; then
    echo "${C_DATE} The memcached is problem and restart!" >> ${LOG_FILE}
    #${MC_CMD} -d -u nobody -m ${MC_MEMORY} -p ${MC_PORT} 
else
    echo "${C_DATE} The memcached is ok!" >> ${LOG_FILE}
fi
```

添加执行权限:
`chmod +x /opt/sh/memcached_check.sh`

添加crontab计划任务,每5分钟检测一次.
`vi /etc/crontab`
```bash
*/5 * * * * root /opt/sh/memcached_check.sh
```
