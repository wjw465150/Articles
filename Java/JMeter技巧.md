
#### [x] 非GUI模式下运行Jmeter
具体方法是：先在GUI模式下创建TestPlan，保存为jmx文件。命令行启动jmeter：./ApacheJMeter -n -t testplan.jmx (选项-n表示non-GUI，-t指定TestPlan文件)。运行结束后Aggregate Report和PerfMon Metrics Collector就会保存在你指定的文件中。把保存PerfMon Metrics Collector的文件拖到Jmerter GUI中就可以看到CUP等使用状况拆线图了。

#### [x] 远程启动jmeter  

应用进场景：用一台机器（称为JMeter客户端）上的jmeter同时启动另外几台机器（称为JMeter远程服务器）上的jmeter。

1. 保证jmeter客户端和jmeter远程服务器采用相同版本的jmeter和JVM。  
2. jmeter客户端和jmeter远程服务器最好在同一个网段内。  
3. 在jmeter远程服务器上运行JMETER_HOME/bin/jmeter-server   （UNIX）或者JMETER_HOME/bin/jmeter-server.bat（Windows）脚本 。  
4. 在jmeter客户端上修改/bin/jmeter.properties文件，找到属性"remote_hosts"，使用JMeter远程服务器的IP地址作为其属性值。可以添加多个服务器的IP地址，以逗号作为分隔。 
例如：  
>  #remote_hosts=127.0.0.1  
>  remote_hosts=9.115.210.2:1099,9.115.210.3:1099,9.115.210.4:1099  
>  # RMI port to be used by the server (must start rmiregistry with same port)  
>  server_port=1099  

5. 在jmeter客户端上启动jmeter:  
`./jmeter -n -t plan.jmx -r #选项-r表示远程启动(remote)`  
jmeter客户端会自动向jmeter远程服务器上分发测试计划。

#### PerfMon用来监控Server的CPU、I/O、Memory等情况。

  1. 插件下载地址：http://code.google.com/p/jmeter-plugins/wiki/PerfMon  
  2. 把JMeterPlugins.jar放到jmeter客户端的jmeter/lib/ext下。  
  3. 启动jmeter，添加Listener时你就看到PerfMon Metrics Collectors了。  
  4. 另外还需要把下载下来的PerfMon解压后放到所有的被测试的服务器上，并运JMeterPlugins/serverAgent/startAgent.sh，默认工作在4444端口。  
  5. 使用PerfMon截图：  
  再次提醒一下，在非GUI模式下运行Jmeter时指定把result保存到一个文件是非常必要的。  
