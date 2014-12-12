#### [X] ��ʱ���Memcached�����Ƿ����,���������Զ�������
----------
����ű����Զ����Memcached�Ľ���,����ҵ����Զ�����Memcached����.
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

#��netstat����鿴memcached����
MM=`netstat -lnp | grep "memcached" | grep "${MC_PORT}" | grep -v "grep" |wc -l`

#if����жϽ����Ƿ����,���������,�����־��¼������memcached����
if [ "$MM" -eq "0" ]; then
    echo "${C_DATE} The memcached is problem and restart!" >> ${LOG_FILE}
    #${MC_CMD} -d -u nobody -m ${MC_MEMORY} -p ${MC_PORT} 
else
    echo "${C_DATE} The memcached is ok!" >> ${LOG_FILE}
fi
```

���ִ��Ȩ��:
`chmod +x /opt/sh/memcached_check.sh`

���crontab�ƻ�����,ÿ5���Ӽ��һ��.
`vi /etc/crontab`
```bash
*/5 * * * * root /opt/sh/memcached_check.sh
```
