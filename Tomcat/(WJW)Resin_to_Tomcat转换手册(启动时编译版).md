## (WJW)Resin_to_Tomcat转换手册(启动时编译版)

#### [x] `/etc/hosts`文件里添加:
> 192.168.1.200   youyuan-baby  
> 192.168.1.200   sqs-youyuan-baby  
> 192.168.1.200   solr-youyuan-baby  

#### [x] 解压`tomcat_youyuan.tar.gz`文件到`/opt/app/`目录下:
> tar zxvf tomcat_youyuan.tar.gz

#### [x] 检查`/opt/app/tomcat_youyuan/conf/`目录里是否有以下3个文件:
> db_pool.properties  
> logging.properties  
> resin.xml  

#### [x] 编辑:`/opt/app/tomcat_youyuan/conf/server.xml`文件,添加:
```xml
        <Context displayName="youyuan" docBase="/opt/app/htdocs/" path="" sessionCookieDomain=".youyuan.com"/>
```
#### [x] 编辑:`/opt/app/tomcat_youyuan/conf/web.xml`文件,修改: `checkInterval为0`
```xml
        <init-param>
            <param-name>checkInterval</param-name>
            <param-value>0</param-value>
        </init-param>
```

#### [x] 把有缘的webapp文件复制到`/opt/app/htdocs/`目录下

#### [x]  复制文件: `wjw-hessian-4.0.38.jar,wjw-ant-jsp.jar`到`htdocs/WEB-INF/lib/`目录下

#### [x] `htdocs/WEB-INF/web.xml`文件里
* 替换: `<web-app xmlns="http://caucho.com/ns/resin"         xmlns:resin="http://caucho.com/ns/resin/core">`
为:
```xml
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://java.sun.com/xml/ns/javaee
                      http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
  version="3.0">
```
* 添加:
```xml
	<listener>
		<listener-class>wjw.jsp.JspCompileListener</listener-class>
	</listener>
```
* 查找:	`<url-pattern>*/FileUploadServlet</url-pattern>`  
替换成: `<url-pattern>/FileUploadServlet</url-pattern>`
