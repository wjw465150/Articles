# (WJW)Docker私有Registry在CentOS6.X下安装指南
**说明:**

> `docker.yy.com` 这是docker registry服务器的域名也就是你的公司docker私有服务器的主机地址，假定ip是`192.168.2.114`;因为https的SSL证书不能用IP地址，我就随便起了个名字。

> `registry` 服务器作为上游服务器处理docker镜像的最终上传和下载，用的是官方的镜像。

> `nginx 1.4.x` 是一个用nginx作为反向代理服务器


- - -
# [X] Docker Server端配置

## 安装依赖
```bash
yum -y install gcc make file && \
yum -y install tar pcre-devel pcre-staticopenssl openssl-devel httpd-tools
```

## 配置SSL
### (1) 编辑`/etc/hosts`,把`docker.yy.com`的ip地址添加进来,例如:
```
192.168.2.114 docker.yy.com
```

### (2) 生成根密钥
先把
> /etc/pki/CA/cacert.pem  
> /etc/pki/CA/index.txt  
> /etc/pki/CA/index.txt.attr  
> /etc/pki/CA/index.txt.old  
> /etc/pki/CA/serial  
> /etc/pki/CA/serial.old  
  
删除掉!  
```bash
cd /etc/pki/CA/
openssl genrsa -out private/cakey.pem 2048
```

### (3) 生成根证书
```bash
openssl req -new -x509 -key private/cakey.pem -out cacert.pem
```
输出:
```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:beijing
Locality Name (eg, city) [Default City]:beijing
Organization Name (eg, company) [Default Company Ltd]:youyuan
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:docker.yy.com
Email Address []:
```
> 会提示输入一些内容，因为是私有的，所以可以随便输入，最好记住能与后面保持一致,特别是"Common Name"。上面的自签证书cacert.pem应该生成在/etc/pki/CA下。

### (4) 为我们的nginx web服务器生成ssl密钥
```bash
mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl
openssl genrsa -out nginx.key 2048
```
> 我们的CA中心与要申请证书的服务器是同一个，否则应该是在另一台需要用到证书的服务器上生成。

### (5) 为nginx生成证书签署请求
```bash
openssl req -new -key nginx.key -out nginx.csr
```
输出:
```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:beijing
Locality Name (eg, city) [Default City]:beijing
Organization Name (eg, company) [Default Company Ltd]:youyuan
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:docker.yy.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
> 同样会提示输入一些内容，Commone Name一定要是你要授予证书的服务器域名或主机名，challenge password不填。

### (6) 私有CA根据请求来签发证书
```bash
touch /etc/pki/CA/index.txt
touch /etc/pki/CA/serial
echo 00 > /etc/pki/CA/serial
openssl ca -in nginx.csr -out nginx.crt
```
输出:
```
Using configuration from /etc/pki/tls/openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 0 (0x0)
        Validity
            Not Before: Dec  9 09:59:20 2014 GMT
            Not After : Dec  9 09:59:20 2015 GMT
        Subject:
            countryName               = CN
            stateOrProvinceName       = beijing
            organizationName          = youyuan
            commonName                = docker.yy.com
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Comment:
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier:
                5D:6B:02:FF:9E:F8:EA:1B:73:19:47:39:4F:88:93:9F:E7:AC:A5:66
            X509v3 Authority Key Identifier:
                keyid:46:DC:F1:A5:6F:39:EC:6E:77:03:3B:C4:34:03:7E:B8:0A:ED:99:41

Certificate is to be certified until Dec  9 09:59:20 2015 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```
> 同样会提示输入一些内容，选择`y`就可以了!


- - -

## 安装,配置,运行nginx
### (1) 添加组和用户:
```
groupadd www -g 58
useradd -u 58 -g www www
```

### (2) 下载nginx源文件:
```bash
cd /tmp
wget http://nginx.org/download/nginx-1.4.6.tar.gz
cp ./nginx-1.4.6.tar.gz /tmp/
```

### (3) 编译,安装nginx:
```bash
tar zxvf ./nginx-1.4.6.tar.gz
cd ./nginx-1.4.6 && \
  ./configure --user=www --group=www --prefix=/opt/nginx \
  --with-pcre \
  --with-http_stub_status_module \
  --with-http_ssl_module \
  --with-http_addition_module  \
  --with-http_realip_module \
  --with-http_flv_module && \
  make && \
  make install
cd /tmp
rm -rf /tmp/nginx-1.4.6/
rm /tmp/nginx-1.4.6.tar.gz
```

### (4) 生成htpasswd
```
htpasswd -cb /opt/nginx/conf/.htpasswd ${USER} ${PASSWORD}
```

### (5) 编辑`/opt/nginx/conf/nginx.conf`文件
```
#daemon off;

# 使用的用户和组
user  www www;
# 指定工作进程数(一般等于CPU总核数)
worker_processes  auto;

# 指定错误日志的存放路径,错误日志记录级别选项为:[debug | info | notic | warn | error | crit]
error_log  /var/log/nginx_error.log  error;

#指定pid存放的路径
#pid        logs/nginx.pid;

# 指定文件描述符数量
worker_rlimit_nofile 51200;

events {
    # 使用的网络I/O模型,Linux推荐epoll;FreeBSD推荐kqueue
    use epoll;
    # 允许的最大连接数
    worker_connections  51200;
    multi_accept on;
}

http {
  include       mime.types;

  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$upstream_addr"';

  access_log  /var/log/nginx_access.log  main;

  # 服务器名称哈希表的桶大小,该默认值取决于CPU缓存
  server_names_hash_bucket_size 128;
  # 客户端请求的Header头缓冲区大小
  client_header_buffer_size 32k;
  large_client_header_buffers 4 32k;

  # 启用sendfile()函数
  sendfile        on;
  tcp_nopush      on;
  tcp_nodelay     on;

  keepalive_timeout  65;

  upstream registry {
    server 127.0.0.1:5000;
  }

  server {
    listen       443;
    server_name  192.168.2.114;

    ssl                  on;
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    client_max_body_size 0; # disable any limits to avoid HTTP 413 for large image uploads

    # required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
    chunked_transfer_encoding on;

    location / {
      auth_basic "registry";
      auth_basic_user_file /opt/nginx/conf/.htpasswd;

      root   html;
      index  index.html index.htm;

      proxy_pass                  http://registry;
      proxy_set_header  Host           $http_host;
      proxy_set_header  X-Real-IP      $remote_addr;
      proxy_set_header  Authorization  "";

      client_body_buffer_size     128k;
      proxy_connect_timeout       90;
      proxy_send_timeout          90;
      proxy_read_timeout          90;
      proxy_buffer_size           8k;
      proxy_buffers               4 32k;
      proxy_busy_buffers_size     64k;  #如果系统很忙的时候可以申请更大的proxy_buffers 官方推荐*2
      proxy_temp_file_write_size  64k;  #proxy缓存临时文件的大小
    }
    location /_ping {
      auth_basic off;
      proxy_pass http://registry;
    }
    location /v1/_ping {
      auth_basic off;
      proxy_pass http://registry;
    }
  }
}

```

### (6) 验证配置
```
/opt/nginx/sbin/nginx -t
```
输出:
>  
>  nginx: the configuration file /opt/nginx/conf/nginx.conf syntax is ok  
>  nginx: configuration file /opt/nginx/conf/nginx.conf test is successful
  

### (7) 启动nginx:
```
/opt/nginx/sbin/nginx
```

### (8) 验证nginx是否启动:
```
ps -ef | grep -i 'nginx'
```
如下输出就表明nginx一切正常!
```
root     27133     1  0 18:58 ?        00:00:00 nginx: master process /opt/nginx/sbin/nginx
www      27134 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27135 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27136 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27137 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27138 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27139 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27140 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27141 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27142 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27143 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27144 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27145 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27146 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27147 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27148 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27149 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27150 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27151 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27152 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27153 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27154 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27155 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27156 27133  0 18:58 ?        00:00:00 nginx: worker process
www      27157 27133  0 18:58 ?        00:00:00 nginx: worker process
root     27160 42863  0 18:58 pts/0    00:00:00 grep -i nginx
```

- - -

## 配置,运行Docker
### (1) 停止docker
```bash
service docker stop
```

### (2)编辑`/etc/sysconfig/docker`文件,加上如下一行
```
DOCKER_OPTS="--insecure-registry docker.yy.com --tlsverify --tlscacert /etc/pki/CA/cacert.pem"
```

### (3) 启动docker
```bash
service docker start
```
_ _ _
## 下载,配置,运行`registry`image

### (1) 获取Image
```bash
docker pull registry
```

### (2) 运行Image
```bash
mkdir -p /opt/registry
docker run -d -e STORAGE_PATH=/registry -v /opt/registry:/registry -p 127.0.0.1:5000:5000 --name registry registry
```
> 命令稍加解释一下:
>  +  `-p 127.0.0.1:5000:5000` registry 作为上游服务器，这个 5000 端口可以不用映射出来，因为所有的外部访问都是通过前端的nginx来提供，nginx 可以在私有网络访问 registry 。



### (3) 验证registry:
> 用浏览器输入: `https://docker.yy.com`
> 或者:`curl -i -k https://abc:123@docker.yy.com`

服务端的配置就到此完成!

- - -

# [X] Docker客户端配置
## (1) 编辑`/etc/hosts`,把`docker.yy.com`的ip地址添加进来,例如:
```
192.168.2.114 docker.yy.com
```

## (2) 把docker registry服务器端的根证书追加到ca-certificates.crt文件里
```
cat ./cacert.pem >> /etc/pki/tls/certs/ca-certificates.crt
```

## (3) 验证`docker.yy.com`下的registry:
> 用浏览器输入: `https://docker.yy.com`
> 或者:`curl -i -k https://abc:123@docker.yy.com`

## (4) 使用私有registry步骤:
+ 登录: `docker login -u abc -p 123 -e "test@gmail.com" https://docker.yy.com`
+ 给container起另外一个名字: `docker tag centos:centos6 docker.yy.com/centos:centos6`
+ 发布: `docker push docker.yy.com/centos:centos6`

- - -
# [X] Server端,操作私有仓库的步骤:
## 1. 从官方pull下来image!  
`docker push centos:centos6`  

## 2. 查看image的id  
执行`docker images`  
输出:  
```
root@pts/0 # docker images
REPOSITORY                                   TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
centos                                       centos6             25c5298b1a36        8 days ago          215.8 MB
```

## 3. 给image赋予一个私有仓库的tag  
`docker tag 25c5298b1a36 docker.yy.com/centos:centos6`  

## 4. push到私有仓库  
`docker push docker.yy.com/centos:centos6`  

## 5. 查看image  
`docker images`  
输出:  
```
root@pts/0 # docker images
REPOSITORY                                   TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
centos                                       centos6             25c5298b1a36        8 days ago          215.8 MB
docker.yy.com/centos                         centos6             25c5298b1a36        8 days ago          215.8 MB
```

# [X] Client端,操作私有仓库的步骤:  
## 1. 从私有仓库pull下来image!  
```
docker pull docker.yy.com/centos:centos6
```
## 2. 查看image  
`docker images`  
输出:  
```
root@pts/0 # docker images
REPOSITORY                                   TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
docker.yy.com/centos                         centos6             25c5298b1a36        8 days ago          215.8 MB
```


- - -
# 附录:  
## (1) 弊端:  
> server端可以login到官方的Docker Hub,可以pull,push官方和私有仓库!  
> client端只能操作搭设好的私有仓库!  
> 私有仓库不能search!  

## (2) 优点:  
> 所有的build,pull,push操作只能在私有仓库的server端操作,降低企业风险!  

## (3) 当client端`docker login`到官方的`https://index.docker.io/v1/`网站,出现`x509: certificate signed by unknown authority`错误时  
> 重命名根证书! `mv /etc/pki/tls/certs/ca-certificates.crt /etc/pki/tls/certs/ca-certificates.crt.bak`  
> 重启docker服务! `service docker restart`!  
