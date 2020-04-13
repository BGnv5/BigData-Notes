#### 监控端口数据

##### 案例需求
首先，Flume监控本机44444端口，然后通过nc工具向本机44444端口发送消息，最后Flume将监听的数据实时显示在控制台。

##### 需求分析
1. 通过nc工具向本机的44444端口发送数据
2. Flume监控本机的44444端口，通过Flume的source端读取数据
3. Flume将获取的数据通过sink端写出到控制台
##### 实现步骤
1. 将rpm软件包烤入/opt/software文件夹下面，执行rpm软件包安装命令，安装nc工具
```
[root@hadoop100 software]# rpm -ivh nc-1.84-24.el6.x86_64.rpm 
```
2. 判断4444端口是否被占用
```
[root@hadoop100 software]# netstat -tunlp | grep 44444
```
netstat 语法：
参数 | 说明
---|---
--tcp,-t | 显示TCP传输协议的连接状况
--udp,-u | 显示UDP传输协议的连接状况
--numeric,-n | 直接使用IP地址，而不通过域名服务器
--listening,-l | 显示监控中的服务器的Socket
--programs,-p | 显示正在使用Socket的程序识别码和程序名称

3. 创建Flume Agent配置文件flume-nc-logger.conf    

在flume目录下创建job文件夹并进入job文件夹
```
[root@hadoop100 flume-1.6.0]# mkdir job
[root@hadoop100 flume-1.6.0]# cd job
```
在job文件夹下创建Flume Agent配置文件flume-nc-logger.conf,并添加如下内容
```
[root@hadoop100 job]# touch flume-nc-logger.conf
[root@hadoop100 job]# vim flume-nc-logger.conf 
# Name the components on this agent      # a1表示agent的名称
a1.sources = r1                          # r1表示a1的输入源
a1.sinks = k1                            # k1表示a1的输出目的地
a1.channels = c1                         # c1表示a1的缓冲区

# Describe/configure the source
a1.sources.r1.type = netcat              # 表示a1的输入源类型为netcat端口类型
a1.sources.r1.bind = localhost           # 表示a1监听的主机
a1.sources.r1.port = 44444               # 表示a1监听的端口号

# Describe the sink                     
a1.sinks.k1.type = logger                # 表示a1的输出目的地是控制台logger类型

# Use a channel which buffers events in memory
a1.channels.c1.type = memory             # 表示a1的channel类型是memory内存型
a1.channels.c1.capacity = 1000           # 表示a1的channel总容量为1000个event
a1.channels.c1.transactionCapacity = 100 # 表示a1的channel传输时收集到了100个event后再去提交事务

# Bind the source and sink to the channel
a1.sources.r1.channels = c1              # 表示将r1和c1连接起来
a1.sinks.k1.channel = c1                 # 表示将k1和c1连接起来
```
4. 开启flume监听
```
[root@hadoop100 job]# flume-ng agent -n a1 -c ./ -f ./flume-nc-logger.conf -Dflume.root.logger=INFO,console
```
启动命令：
参数 | 描述
---|---
agent | 运行一个Flume Agent
--conf,-c <conf> | 指定配置文件放在什么目录
--conf-file,-f <file> (必填)| 指定配置文件，这个配置文件必须在全局选项的--conf参数定义的目录下
--name,-n <name> (必填)| Agent的名字，注意：要和配置文件里的名字一致
-Dproperty=value | 设置一个JAVA系统属性值。常见的：-Dflume.root.logger=INFO,console,-D表示flume运行时动态修改flume.root.logger参数属性值，并将控制台日志打印级别设置为INFO级别，日志级别包括：log、info、warn、error

5. 通过nc工具向本机的44444端口发送内容
```
[root@hadoop100 ~]# nc localhost 44444
hello flume
OK
```
或者通过外部http请求访问对应的IP和端口，如：http://192.168.1.100:44444/hello

6. 在Flume监听页面观察接收数据情况
```
2020-04-12 13:11:48,650 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:94)] Event: { headers:{} body: 68 65 6C 6C 6F 20 66 6C 75 6D 65                hello flume }
```
#### 实时读取本地文件到hdfs

##### 案例需求
实时监控Hive日志，并上传到HDFS中

##### 需求分析
1. 创建符合条件的flume配置文件
2. 执行配置文件，开启监控
3. 开启Hive,生成日志
4. 查看HDFS上数据

##### 实现步骤
1. Flume要想将数据输出到HDFS，必须持有Hadoop相关jar包
```
commons-configuration-1.6.jar
hadoop-auth-2.7.1.jar
hadoop-comment-2.7.1.jar
hadoop-hdfs-2.7.1.jar
hadoop-marreduce-client-core-2.7.1.jar
```
2. 创建flume-file-hdfs.conf文件,并添加如下内容：

```
[root@hadoop100 job]# touch flume-file-hdfs.conf
[root@hadoop100 job]# vim flume-file-hdfs.conf
# Name the components on this agent
a2.sources = r2
a2.sinks = k2
a2.channels = c2

# Describe/configure the source
a2.sources.r2.type = exec
a2.sources.r2.command = tail -F /tmp/root/hive.log

# Describe the sink
a2.sinks.k2.type = hdfs
a2.sinks.k2.hdfs.path=hdfs://hadoop100:9000/flume/%Y%m%d/%H
a2.sinks.k2.hdfs.useLocalTimeStamp = true

# Use a channel which buffers events in memory
a2.channels.c2.type = memory
a2.channels.c2.capacity = 1000
a2.channels.c2.transactionCapacity = 100

# Bind the source and sink to the channel
a2.sources.r2.channels = c2
a2.sinks.k2.channel = c2
```
3. 执行监控配置
```
[root@hadoop100 job]# flume-ng agent -n a2 -f ./flume-file-hdfs.conf 
```
4. 开启Hadoop和Hive并操作hive产生日志
```
[root@hadoop100 ~]# start-all.sh
[root@hadoop100 ~]# hive
```
5. 在HDFS上查看文件
```
[root@hadoop100 bin]# hadoop dfs -cat /flume/20200412/14/FlumeData.1586674705784
```
#### 实时读取目录文件到HDFS

##### 案例需求
使用Flume监控整个目录的文件

##### 需求分析
1. 创建符合条件的flume配置文件
2. 执行配置文件，开启监控
3. 向upload目录中添加文件
4. 查看HDFS上数据
5. 查看/opt/module/flume/upload目录中上传的文件是否已经标记为.COMPLETED结尾；.tmp后缀结尾文件没有上传

##### 实现步骤
1. 创建配置文件flume-dir-hdfs.conf,并添加如下内容
```
a3.sources=r3
a3.sinks=k3
a3.channels=c3

#Describe/configure the source
a3.sources.r3.type=spooldir
a3.sources.r3.spoolDir=/opt/module/flume/upload
a3.sources.r3.fileSuffix=.COMPLETED
a3.sources.r3.fileHeader=true

#忽略所有以.tmp结尾的文件，不上传
a3.sources.r3.ignorePattern=([^]*\.tmp)

# Describe the sink
a3.sinks.k3.type=hdfs
a3.sinks.k3.hdfs.path=hdfs://hadoop100:9000/flume/upload/%Y%m%d/%H
#是否使用本地时间戳
a3.sinks.k3.hdfs.useLocalTimeStamp=true

#上传文件的前缀
a3.sinks.k3.hdfs.filePrefix=upload-

#是否按照时间滚动文件夹
a3.sinks.k3.hdfs.round=true

#多少时间单位创建一个新的文件夹
a3.sinks.k3.hdfs.roundValue=1

#重新定义时间单位
a3.sinks.k3.hdfs.roundUnit=hour

#积攒多少个Event才flush到HDFS一次
a3.sinks.k3.hdfs.batchSize=100

#设置文件类型，可支持压缩
a3.sinks.k3.hdfs.fileType=DataStream

#多久生成一个新的文件
a3.sinks.k3.hdfs.rollInterval=600

#设置每个文件的滚动大小大概是128M
a3.sinks.k3.hdfs.rollSize=134217700

#文件的滚动与Event数量无关
a3.sinks.k3.hdfs.rollCount=0

# 最小冗余数
a3.sinks.k3.hdfs.minBlockReplicas=1

# Useachannelwhichbufferseventsinmemory
a3.channels.c3.type=memory
a3.channels.c3.capacity=1000
a3.channels.c3.transactionCapacity=100

# Bindthesourceandsinktothechannel
a3.sources.r3.channels=c3
a3.sinks.k3.channel=c3
```
2. 启动监控文件夹命令
```
[root@hadoop100 job]# flume-ng agent -n a3 -f ./flume-dir-hdfs.conf 
```
说明：在使用Spooling Directory Source时

（1）不要在监控目录中创建并持续修改文件    
（2）上传完成的文件会以.COMPLETED结尾   
（3）被监控文件夹每500毫秒扫描一次文件变动
3. 向upload文件夹中添加文件
```
[root@hadoop100 flume-1.6.0]# mkdir load
[root@hadoop100 upload]# touch bgnv5.txt
[root@hadoop100 upload]# touch bgnv5.tmp
[root@hadoop100 upload]# touch bgnv5.log
```
4. 查看HDFS上的数据
```
[root@hadoop100 ~]# hadoop dfs -ls /flume/upload/20200412/19
```
5. 查看upload文件夹
```
[root@hadoop100 upload]# ls
bgnv5.log.COMPLETED  bgnv5.tmp  bgnv5.txt.COMPLETED
```
#### 单数据源多出口（选择器）

![多路复用流](pics\DLFY.jpg)

##### 案例需求
使用flume-1监控文件变动，flume-1将变动内容传递给flume-2，flume-2负责存储到HDFS。同时flume-1将变动内容传递给flume-3，flume-3负责输出到local filesystem。

##### 实现步骤
1. 准备工作

在/opt/module/flume-1.6.0/job目录下创建group1 文件夹
```
[root@hadoop100 group1]# cd /opt/module
```
在/opt/module/datas/目录下创建flume3文件夹
```
[root@hadoop100 datas]# mkdir flume3
```
2. 创建flume-file-flume.conf，并添加如下内容

配置1个接收日志文件的source和两个channel、两个sink，分别输送给flume-flume-hdfs和flume-flume-dir。
```
[root@hadoop100 group1]$ touch flume-file-flume.conf
[root@hadoop100 group1]$ vim flume-file-flume.conf
# Name the components on this agent
a1.sources = r1
a1.sinks = k1 k2
a1.channels = c1 c2
 
# 将数据流复制给多个channel
a1.sources.r1.selector.type = replicating
 
# Describe/configure the source
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /tmp/root/hive.log
a1.sources.r1.shell = /bin/bash -c
 
# Describe the sink
a1.sinks.k1.type = avro
a1.sinks.k1.hostname = hadoop100
a1.sinks.k1.port = 4141

a1.sinks.k2.type = avro
a1.sinks.k2.hostname = hadoop100
a1.sinks.k2.port = 4142
 
# Describe the channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

a1.channels.c2.type = memory
a1.channels.c2.capacity = 1000
a1.channels.c2.transactionCapacity = 100
 
# Bind the source and sink to the channel
a1.sources.r1.channels = c1 c2
a1.sinks.k1.channel = c1
a1.sinks.k2.channel = c2
```
3. 创建flume-flume-hdfs.conf，并添加如下内容
配置上级Flume输出的Source,输出是到HDFS的sink
```
[root@hadoop100 group1]$ touch flume-flume-hdfs.conf
[root@hadoop100 group1]$ vim flume-flume-hdfs.conf
# Name the components on this agent
a2.sources = r1
a2.sinks = k1
a2.channels = c1
 
# Describe/configure the source
a2.sources.r1.type = avro
a2.sources.r1.bind = hadoop100
a2.sources.r1.port = 4141
 
# Describe the sink
a2.sinks.k1.type = hdfs
a2.sinks.k1.hdfs.path = hdfs://hadoop100:9000/flume2/%Y%m%d/%H
#是否使用本地时间戳
a2.sinks.k1.hdfs.useLocalTimeStamp = true

#上传文件的前缀
a2.sinks.k1.hdfs.filePrefix = flume2-
 
#是否按照时间滚动文件夹
a2.sinks.k1.hdfs.round = true
 
#多少时间单位创建一个新的文件夹
a2.sinks.k1.hdfs.roundValue = 1
 
#重新定义时间单位
a2.sinks.k1.hdfs.roundUnit = hour

#积攒多少个Event才flush到HDFS一次
a2.sinks.k1.hdfs.batchSize = 100
 
#设置文件类型，可支持压缩
a2.sinks.k1.hdfs.fileType = DataStream
 
#多久生成一个新的文件
a2.sinks.k1.hdfs.rollInterval = 600
 
#设置每个文件的滚动大小大概是128M
a2.sinks.k1.hdfs.rollSize = 134217700
 
#文件的滚动与Event数量无关
a2.sinks.k1.hdfs.rollCount = 0
 
#最小冗余数
a2.sinks.k1.hdfs.minBlockReplicas = 1
 
# Describe the channel
a2.channels.c1.type = memory
a2.channels.c1.capacity = 1000
a2.channels.c1.transactionCapacity = 100
 
# Bind the source and sink to the channel
a2.sources.r1.channels = c1
a2.sinks.k1.channel = c1
```
3. 创建flume-flume-dir.conf,并添加如下内容

配置上级flume输出的source，输出是到本地目录的sink。
```
[root@hadoop100 group1]$ touch flume-flume-dir.conf
[root@hadoop100 group1]$ vim flume-flume-dir.conf
# Name the components on this agent
a3.sources = r1
a3.sinks = k1
a3.channels = c2
 
# Describe/configure the source
a3.sources.r1.type = avro
a3.sources.r1.bind = hadoop100
a3.sources.r1.port = 4142
 
# Describe the sink
a3.sinks.k1.type = file_roll
a3.sinks.k1.sink.directory = /opt/module/datas/flume3
 
# Describe the channel
a3.channels.c2.type = memory
a3.channels.c2.capacity = 1000
a3.channels.c2.transactionCapacity = 100
 
# Bind the source and sink to the channel
a3.sources.r1.channels = c2
a3.sinks.k1.channel = c2
```
提示：输出的本地目录必须是已经存在的目录，如果该目录不存在，并不会创建新的目录。
4. 执行配置文件

分别开启对应配置文件：flume-flume-dir，flume-flume-hdfs，flume-file-flume。
```
[root@hadoop100 flume]$ bin/flume-ng agent -n a3 -f job/group1/flume-flume-dir.conf
[root@hadoop100 flume]$ bin/flume-ng agent -n a2 -f job/group1/flume-flume-hdfs.conf
[root@hadoop100 flume]$ bin/flume-ng agent -n a1 -f job/group1/flume-file-flume.conf
```
5. 启动Hadoop和Hive
```
[root@hadoop100 ~]# start-all.sh
[root@hadoop100 ~]# hive
```
6. 检查HDFS上数据
```
[root@hadoop100 ~]# hadoop dfs -ls /flume2/20200412/19
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.

Found 1 items
-rw-r--r--   1 root supergroup       1563 2020-04-12 19:30 /flume2/20200412/19/flume2-.1586690993114.tmp
```
7. 检查/opt/module/datas/flume3目录中的数据
```
[root@hadoop100 flume3]# ll
总用量 8
-rw-r--r-- 1 root root    0 4月  12 19:26 1586690789923-1
```
#### 单数据源多出口（sink组）

![负载均衡功能](pics\FZJH.jpg)

##### 案例需求
使用Flume-1监控文件变动，Flume-1将变动内容传递给Flume-2,Flume-2负责存储到HDFS。同时Flume-1将变动内容传递给Flume-3,Flume-3也负责存储到HDFS

##### 实现步骤

1. 准备工作

在/opt/module/flume/job目录下创建一个group2文件夹
```
[root@hadoop100 job]$ mkdir group2
```
2. 创建flume-file2-flume.conf，并添加如下内容：

配置1个接收日志文件的source和1个channel、两个sink,分别输送给flume-flume-hdfs1和flume-flume-hdfs2

```
[root@hadoop100 group2]$ touch flume-file2-flume.conf
[root@hadoop100 group2]$ vim flume-file2-flume.conf
# Name the components on this agent
a1.sources = r1
a1.channels = c1
a1.sinkgroups = g1
a1.sinks = k1 k2

# Describe/configure the source
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /tmp/root/hive.log
a1.sources.r1.shell = /bin/bash -c

a1.sinkgroups.g1.processor.type = load_balance
a1.sinkgroups.g1.processor.backoff = true
a1.sinkgroups.g1.processor.selector = round_robin
a1.sinkgroups.g1.processor.selector.maxTimeOut=10000

# Describe the sink
a1.sinks.k1.type = avro
a1.sinks.k1.hostname = hadoop100
a1.sinks.k1.port = 4141

a1.sinks.k2.type = avro
a1.sinks.k2.hostname = hadoop100
a1.sinks.k2.port = 4142

# Describe the channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinkgroups.g1.sinks = k1 k2
a1.sinks.k1.channel = c1
a1.sinks.k2.channel = c1
```

3. 创建flume-flume-hdfs1.conf，并添加如下内容
```
[root@hadoop100 group1]$ touch flume-flume-hdfs1.conf
[root@hadoop100 group1]$ vim flume-flume-hdfs1.conf
# Name the components on this agent
a2.sources = r1
a2.sinks = k1
a2.channels = c1

# Describe/configure the source
a2.sources.r1.type = avro
a2.sources.r1.bind = hadoop100
a2.sources.r1.port = 4141

# Describe the sink
a2.sinks.k1.type = hdfs
a2.sinks.k1.hdfs.path = hdfs://hadoop100:9000/flume3/%Y%m%d/%H
#是否使用本地时间戳
a2.sinks.k1.hdfs.useLocalTimeStamp = true

#上传文件的前缀
a2.sinks.k1.hdfs.filePrefix = flume3-
 
#是否按照时间滚动文件夹
a2.sinks.k1.hdfs.round = true
 
#多少时间单位创建一个新的文件夹
a2.sinks.k1.hdfs.roundValue = 1
 
#重新定义时间单位
a2.sinks.k1.hdfs.roundUnit = hour

#积攒多少个Event才flush到HDFS一次
a2.sinks.k1.hdfs.batchSize = 100
 
#设置文件类型，可支持压缩
a2.sinks.k1.hdfs.fileType = DataStream
 
#多久生成一个新的文件
a2.sinks.k1.hdfs.rollInterval = 600
 
#设置每个文件的滚动大小大概是128M
a2.sinks.k1.hdfs.rollSize = 134217700
 
#文件的滚动与Event数量无关
a2.sinks.k1.hdfs.rollCount = 0
 
#最小冗余数
a2.sinks.k1.hdfs.minBlockReplicas = 1

# Describe the channel
a2.channels.c1.type = memory
a2.channels.c1.capacity = 1000
a2.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a2.sources.r1.channels = c1
a2.sinks.k1.channel = c1
```

4. 创建flume-flume-hdfs2.conf，并添加如下内容
```
[root@hadoop100 group1]$ touch flume-flume-hdfs2.conf
[root@hadoop100 group1]$ vim flume-flume-hdfs2.conf
# Name the components on this agent
a3.sources = r1
a3.sinks = k1
a3.channels = c2

# Describe/configure the source
a3.sources.r1.type = avro
a3.sources.r1.bind = hadoop100
a3.sources.r1.port = 4142

# Describe the sink
a2.sinks.k1.type = hdfs
a2.sinks.k1.hdfs.path = hdfs://hadoop100:9000/flume3/%Y%m%d/%H
#是否使用本地时间戳
a2.sinks.k1.hdfs.useLocalTimeStamp = true

#上传文件的前缀
a2.sinks.k1.hdfs.filePrefix = flume3-
 
#是否按照时间滚动文件夹
a2.sinks.k1.hdfs.round = true
 
#多少时间单位创建一个新的文件夹
a2.sinks.k1.hdfs.roundValue = 1
 
#重新定义时间单位
a2.sinks.k1.hdfs.roundUnit = hour

#积攒多少个Event才flush到HDFS一次
a2.sinks.k1.hdfs.batchSize = 100
 
#设置文件类型，可支持压缩
a2.sinks.k1.hdfs.fileType = DataStream
 
#多久生成一个新的文件
a2.sinks.k1.hdfs.rollInterval = 600
 
#设置每个文件的滚动大小大概是128M
a2.sinks.k1.hdfs.rollSize = 134217700
 
#文件的滚动与Event数量无关
a2.sinks.k1.hdfs.rollCount = 0
 
#最小冗余数
a2.sinks.k1.hdfs.minBlockReplicas = 1


# Describe the channel
a3.channels.c2.type = memory
a3.channels.c2.capacity = 1000
a3.channels.c2.transactionCapacity = 100

# Bind the source and sink to the channel
a3.sources.r1.channels = c2
a3.sinks.k1.channel = c2
```
5. 执行配置文件
分别开启对应配置文件：flume-flume-hdfs2.conf，flume-flume-hdfs1.conf，flume-file2-flume.conf。
```
[root@hadoop100 flume]$ bin/flume-ng agent -n a3 -f job/group2/flume-flume-hdfs2.conf
[root@hadoop100 flume]$ bin/flume-ng agent -n a2 -f job/group2/flume-flume-hdfs1.conf
[root@hadoop100 flume]$ bin/flume-ng agent -n a1 -f job/group2/flume-file2-flume.conf
```
5. 启动Hadoop和Hive
```
[root@hadoop100 ~]# start-all.sh
[root@hadoop100 ~]# hive
```
6. 检查HDFS上数据
```
[root@hadoop100 ~]# hadoop dfs -ls /flume3/20200412/20
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.

Found 1 items
-rw-r--r--   1 root supergroup       1563 2020-04-12 20:47 /flume3/20200412/20/flume3-.1586695663649.tmp
```

#### 多数据源汇总

![流的合并](pics\LDHB.jpg)

##### 案例需求

flume-1监控文件hive.log，flume-2监控某一个端口的数据流，flume-1与flume-2将数据发送给flume-3，flume3将最终数据写入到HDFS

##### 实现步骤

1. 准备工作

分发Flume
```
[root@hadoop100 module]$ xsync flume-1.6.0
```
在hadoop101,hadoop102以及hadoop103的/opt/module/flume/job目录下创建一个group3文件夹
```
[root@hadoop101 job]$ mkdir group3
[root@hadoop102 job]$ mkdir group3
[root@hadoop103 job]$ mkdir group3
```
2. 在hadoop102上创建flume-file-flume.conf并添加如下内容

配置source用于监控hive.log文件，配置sink输出数据到下一级flume。
```
[root@hadoop102 group3]$ touch flume-file-flume.conf
[root@hadoop102 group3]$ vim flume-file-flume.conf
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1
 
# Describe/configure the source
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /tmp/root/hive.log
a1.sources.r1.shell = /bin/bash -c
 
# Describe the sink
a1.sinks.k1.type = avro
a1.sinks.k1.hostname = hadoop103
a1.sinks.k1.port = 4141
 
# Describe the channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
 
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```
3. 在hadoop101上创建flume-netcat-flume.conf并添加如下内容

配置source监控端口44444数据流，配置sink数据到下一级flume
```
[root@hadoop101 group3]$ touch flume-netcat-flume.conf
[root@hadoop101 group3]$ vim flume-netcat-flume.conf
# Name the components on this agent
a2.sources = r1
a2.sinks = k1
a2.channels = c1
 
# Describe/configure the source
a2.sources.r1.type = netcat
a2.sources.r1.bind = hadoop101
a2.sources.r1.port = 44444
 
# Describe the sink
a2.sinks.k1.type = avro
a2.sinks.k1.hostname = hadoop103
a2.sinks.k1.port = 4141
 
# Use a channel which buffers events in memory
a2.channels.c1.type = memory
a2.channels.c1.capacity = 1000
a2.channels.c1.transactionCapacity = 100
 
# Bind the source and sink to the channel
a2.sources.r1.channels = c1
a2.sinks.k1.channel = c1
```
4. 在hadoop103上创建flume-flume-hdfs.conf,并添加如下内容

配置source用于接收flume-file-flume与flume-netcat-flume发送过来的数据流，最终合并后sink到HDFS。
```
[root@hadoop103 group3]$ touch flume-flume-hdfs.conf
[root@hadoop103 group3]$ vim flume-flume-hdfs.conf
# Name the components on this agent
a3.sources = r1
a3.sinks = k1
a3.channels = c1
 
# Describe/configure the source
a3.sources.r1.type = avro
a3.sources.r1.bind = hadoop103
a3.sources.r1.port = 4141
 
# Describe the sink
a3.sinks.k1.type = hdfs
a3.sinks.k1.hdfs.path = hdfs://hadoop103:9000/flume4/%Y%m%d/%H
#是否使用本地时间戳
a3.sinks.k1.hdfs.useLocalTimeStamp = true
 
#上传文件的前缀
a3.sinks.k1.hdfs.filePrefix = flume4-
 
#是否按照时间滚动文件夹
a3.sinks.k1.hdfs.round = true
 
#多少时间单位创建一个新的文件夹
a3.sinks.k1.hdfs.roundValue = 1
 
#重新定义时间单位
a3.sinks.k1.hdfs.roundUnit = hour
 
#积攒多少个Event才flush到HDFS一次
a3.sinks.k1.hdfs.batchSize = 100
 
#设置文件类型，可支持压缩
a3.sinks.k1.hdfs.fileType = DataStream
 
#多久生成一个新的文件
a3.sinks.k1.hdfs.rollInterval = 600
 
#设置每个文件的滚动大小大概是128M
a3.sinks.k1.hdfs.rollSize = 134217700
 
#文件的滚动与Event数量无关
a3.sinks.k1.hdfs.rollCount = 0
 
#最小冗余数
a3.sinks.k1.hdfs.minBlockReplicas = 1
 
# Describe the channel
a3.channels.c1.type = memory
a3.channels.c1.capacity = 1000
a3.channels.c1.transactionCapacity = 100
 
# Bind the source and sink to the channel
a3.sources.r1.channels = c1
a3.sinks.k1.channel = c1
```
5. 执行配置文件

分别开启对应配置文件：flume-flume-hdfs.conf，flume-netcat-flume.conf，flume-file-flume.conf。
```
[root@hadoop103 flume]$ bin/flume-ng agent  -n a3 -f job/group3/flume-flume-hdfs.conf
[root@hadoop101 flume]$ bin/flume-ng agent -n a2 -f job/group3/flume-netcat-flume.conf
[root@hadoop102 flume]$ bin/flume-ng agent -n a1 -f job/group3/flume-file-flume.conf
```
6.  启动hadoop和hive
```
[root@hadoop102 hadoop-2.7.2]$ start-all.sh
[root@hadoop102 hive]$ bin/hive
```
7. 向44444端口发送数据
```
[root@hadoop102 flume]$ nc hadoop102 44444
```
8. 检查HDFS上数据
```
[root@hadoop100 ~]# hadoop dfs -ls /flume4/20200412/20
```
