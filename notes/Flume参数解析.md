#### 一、启动命令

参数 | 描述
---|---
agent（必填） | 运行一个Flume Agent
--conf,-c <conf> | 指定配置文件放在什么目录
--conf-file,-f <file> （必填）| 指定配置文件，这个配置文件必须在全局选项的--conf参数定义的目录下
--name,-n <name> （必填）| Agent的名字，注意：要和配置文件里的名字一致
-Dproperty=value | 设置一个JAVA系统属性值。常见的：-Dflume.root.logger=INFO,console

#### 二、Source

##### NetCat Source

一个NetCat Source用来监听一个指定端口，并接收监听到的数据，接收的数据是字符串形式

配置项 | 说明
---|---
channels （必填）| 绑定通道
type (必填) | netcat
bind （必填）| 需要监听的主机名或IP地址
port （必填）| 要监听的端口号
selector.*  | 选择器配置
interceptors.* | 拦截器配置

```
a1.sources.r1.type=netcat
a1.sources.r1.bind=0.0.0.0
a1.sources.r1.port=44444
```

##### Avro Source

通过监听Avro端口来接收外部avro客户端发送的日志信息，avro-source接收到的是经过avro序列化后的数据，然后反序列化数据继续传输。源数据必须是经过avro序列化后的数据。利用avro source可以实现多级流动、扇出流、扇入流等效果。

配置项 | 说明
---|---
channels （必填）| 绑定通道
type (必填) | avro
bind （必填）| 需要监听的主机名或IP地址
port （必填）| 要监听的端口号
threads | 工作线程最大线程数
selector.*  | 选择器配置
interceptors.* | 拦截器配置

```
a1.sources.r1.type=avro
a1.sources.r1.bind=0.0.0.0
a1.sources.r1.port=44444
```

##### Exec Source

可以将命令产生的输出作为源来进行传递

配置项 | 说明
---|---
channels （必填）| 绑定通道
type (必填) | exec
command(必填) | 要执行的命令
shell | 运行命令的shell脚本
selector.*  | 选择器配置
interceptors.* | 拦截器配置

```
a1.sources.r1.type=exec
a1.sources.r1.command=tail -F /tmp/root/hive.log
a1.sources.r1.shell=/bin/bash -c
```

##### Spooling Directory Source

flume会持续监听指定的目录，把放入这个目录中的文件当做source来处理

注意：
- 一旦文件被放到"自动收集"目录后，变不能修改，如果修改，flume会报错。
- 此外，也不能有重名的文件，如果有，flume也会报错

配置项 | 说明
---|---
channels （必填）| 绑定通道
type (必填) | spooldir
spoolDir （必填）| 读被监控的文件夹目录
selector.*  | 选择器配置
interceptors.* | 拦截器配置

```
a1.sources.r1.type=spooldir
a1.sources.r1.spoolDir=/home/work/data
```

##### HTTP Source

此source接收Http的GET和POST请求作为flume的事件，如果想让flume正确解析http协议信息，比如解析出请求头、请求体等信息，需要提供一个可插拔的“处理器”来将请求转化为事件对象，这个处理器必须实现HTTPSourceHeadler接口，这个处理器接收一个HttpServletRequest对象，并返回一个Flume event对象集合。

常用的Handler

- JSONHandler：可以处理JSON格式的数据，并支持UTF-8(默认),UTF-16,UTF-32字符集，该handler接受event数组，并根据请求头中指定的编码将其转化为Flume Event。
- BlobHandler：一种将请求中上传文件信息转化为event的处理器，适合大文件的传输

配置项 | 说明
---|---
channels （必填）| 绑定通道
type (必填) | http
port（必填）| 端口
selector.*  | 选择器配置
interceptors.* | 拦截器配置

```
a1.sources.r1.type=http
a1.sources.r1.port=8888
a1.sources.r1.bind=192.168.1.100

执行curl命令，可以模拟http请求：curl -X POST -d '[{"headers":{"a":"a1","b":"b1"},"body":"hello http flume"}]' http://192.168.1.100:8888
```

#### 三、Channels

##### Memory Channel

事件将被存储在内存中（指定大小的队列里），非常适合那些需要高吞吐量且允许数据丢失的场景下

配置项 | 说明
---|---
type (必填) | memory
capacity | 存储在channel中的最大事件数，建议实际工作调节：10万
transactionCapacity | 每一个事务中的最大事件数，建议实际工作调节：1000~3000

```
a1.channels.c1.type=memory
a1.channels.c1.capacity=1000
a1.channels.c1.transactionCapacity=100
```

##### File Channel

将数据临时存储到计算机的磁盘的文件中，性能比较低，但是即使程序出错数据也不会丢失

配置项 | 说明
---|---
type (必填) | file
dataDirs（必填） | 指定存放的目录

```
a1.channels.c1.type=file
a1.channels.c1.dataDirs=/home/filechannel
```

#### 四、Sink

##### Logger Sink

记录指定级别（如INFO,DEBUG,ERROR）的日志,通常用于调试,根据设计logger sink 将body内容限制为16字节，从而避免屏幕充斥着过多的内容。如果想要查看调试的完整内容，可以使用其他sink，如file_roll sink 它会将日志写到本地文件系统中。

配置项 | 说明
---|---
channels （必填）| 绑定通道
type (必填) | logger

```
a1.sinks.s1.type=logger
```

##### HDFS Sink

此sink将事件写入到hadoop分布式文件系统HDFS中，目前支持text 和 sequence files两种文件格式，支持压缩，并可以对数据进行分区，分桶存储。

配置项 | 说明
---|---
channels （必填）| 绑定通道
type (必填) | hdfs
hdfs.path（必填）| HDFS目录路径（如：hdfs://namenode/flume/webdata）
hdfs.fileType | 文件格式（SequenceFile/DataStream/CompressedStream）
hdfs.codeC | 文件压缩格式（gzip,bzip2,lzo,lzop,snappy）,文件格式为CompressedStream时必须指定
hdfs.filePrefix | 上传文件的前缀
hdfs.fileSuffix | 上传文件的后缀
hdfs.inUsePrefix | Flume正在处理的文件所加的前缀
hdfs.inUseSuffix | Flume正在处理的文件所加的后缀,默认.tmp
hdfs.round | true 是否按照时间滚动文件夹
hdfs.roundValue | 多少时间单位创建一个新的文件夹，默认为1
hdfs.roundUnit | 重新定义时间单位 （second,minute,hour）,默认second
hdfs.timeZone | 时区，默认为Local Time
hdfs.useLocalTimeStamp | 是否使用本地时间戳，默认为false
hdfs.rollInterval | 文件生成的时间间隔（秒），默认是30秒,为0 表示不根据时间来滚动文件
hdfs.rollSize | 生成的文件大小（字节），默认是1024个字节，为0表示不根据文件大小来滚动文件
hdfs.rollCount | 每写几条数据就生成一个新文件，默认数量为10，为0表示不根据event数量来滚动文件
hdfds.batchSize | 每个批次刷新到HDFS上的events数量
hdfs.maxOpenFiles | 最大允许打开的HDFS文件数，默认5000，超过这个值时，最早打开的文件会被关闭
hdfs.minBlockReplicas | HDFS副本数，写入HDFS文件块的最小副本数，该值会影响文件的滚动配置，一般为1，才可以按照配置正确滚动文件
```
a1.sinks.s1.type=hdfs
a1.sinks.s1.hdfs.path=hdfs://192.168.1.100:9000/flume
a1.sinks.s1.hdfs.fileType = DataStream
a1.sinks.s1.hdfs.filePrefix = logs-
```

##### File_roll Sink

在本地系统中存储事件，每隔指定时长生成文件保存这段时间内收集到的日志信息

配置项 | 说明
---|---
channels （必填）| 绑定通道
type (必填) | file_roll
sink.directory （必填）| 文件被存储的目录
sink.rollInterval | 每隔几秒生成一个新的日志文件。如果为0，则禁止滚动，从而导致所有数据都被写到同一个文件中

```
a1.sinks.s1.type=file_roll
a1.sinks.s1.sink.directory=/home/work/rolldata
a1.sinks.s1.sink.rollInterval=60
```

##### Avro Sink

将源数据利用avro进行序列化之后写到指定的节点上，是实现多级流动、扇出流、扇入流的基础

配置项 | 说明
---|---
channels （必填）| 绑定通道
type (必填) | avro
hostname（必填）| 要发送的主机
port（必填）| 要发往的端口号

```
a1.sinks.s1.type=avro
a1.sinks.s1.hostname=192.168.1.100
a1.sinks.s1.port=4141
```

#### 五、Selector

##### Replicating  Selector 

复制模式，Selector默认的模式，当source接收到数据后，会复制多分，分发给每一个avro sink

配置项 | 说明
---|---
selector.type | replicating

##### Multiplexing Selector

多路复用模式，用户可以指定转发的规则。selector根据规则进行数据的分发

配置项 | 说明
---|---
selector.type | multiplexing 表示路由模式
selector.header | 指定要监测的头的名称
selector.mapping.* | 匹配规则
selector.sefault | 如果未满足匹配规则，则默认发往指定的通道

```
a1.sources.r1.selector.type=multiplexing
a1.sources.r1.selector.header=state
a1.sources.r1.selector.mapping.cn=c1
a1.sources.r1.selector.mapping.us=c2
a1.sources.r1.selector.mapping.default=c2 

测试：curl -X POST -d '[{"headers":{"state":"jp"},"body":"hello flume"}]' http://0.0.0.0:8888
```

#### 六、Interceptor

##### Timestamp Interceptor
 
这个拦截器在事件头中插入以毫秒为单位的当前处理时间，头的名字为timestamp，值为当前处理的时间戳，如果在之前已经有这个时间戳，则保留原有的时间戳

配置项 | 说明
---|---
type (必填) | timestamp
preserveExisting | false 如果时间戳已经存在是否保留

```
a1.sources.r1.interceptors=i1
a1.sources.r1.interceptors.i1.type=timestamp

Event:{headers:{state=jp,timestamp:1472280180424} body: 69 64 6F 61 6C 2E 6F 72 67 5F 62 6F 64 79 idoall.org_body}
```

##### Host Interceptor

这个拦截器插入当前处理Agent的主机名或ip,头的名字为host或者配置的名称，值为主机名或IP地址，基于配置

配置项 | 说明
---|---
type (必填) | host
preserveExisting | false 如果主机名已经存在是否保留
useIP | true 如果配置为true则用IP，为false则用主机名
hostHeader | host 加入头时使用的名称

```
a1.sources.r1.interceptors=i1
a1.sources.r1.interceptors.i1.type=host

Event:{headers:{ host=127.0.0.1,state=jp} body: 69 64 6F 61 6C 2E 6F 72 67 5F 62 6F 64 79 idoall.org_body}
```

##### Static Interceptor

此拦截器允许用户增加静态头信息使用静态的值到所有事件，目前的实现中不允许一次指定多个头，如果需要增加多个静态头可以指定多个Static interceptors

配置项 | 说明
---|---
type (必填) | static
key （必填）| key 要增加的头名
value（必填）|value 要增加的头值
preserveExisting | true 

```
a1.sources.r1.interceptors=i1
a1.sources.r1.interceptors.i1.type=static
a1.sources.r1.interceptors.i1.key=addr
a1.sources.r1.interceptors.i1.value=beijing

Event:{headers:{ host=127.0.0.1,state=jp,addr=beijing} body: 69 64 6F 61 6C 2E 6F 72 67 5F 62 6F 64 79 idoall.org_body}
```

##### UUID Interceptor

这个拦截器在所有事件头中增加一个全局一致性标志，其实就是UUID

配置项 | 说明
---|---
type (必填) | org.apache.flume.sink.solr.morphine.UUIDInterceptor$Builder
headerName | id 头名称
preserveExisting | true 如果头已经存在是否保留
prefix | "" 在UUID前拼接的字符串前缀

```
a1.sources.r1.interceptors=i1
a1.sources.r1.interceptors.i1.type=org.apache.flume.sink.solr.morphine.UUIDInterceptor$Builder

Event:{headers:{ host=127.0.0.1,state=jp,addr=beijing,id=d354d2f0-14ff-4815-b382-99ae0a191926} body: 69 64 6F 61 6C 2E 6F 72 67 5F 62 6F 64 79 idoall.org_body}
```

##### Search And Replace Interceptor

这个拦截器提供了简单的基于字符串的正则搜索和替换功能

配置项 | 说明
---|---
type (必填) | search_replace
searchPattern（必填）| 要搜索和替换的正则表达式
replaceString（必填）| 要替换为的字符串
charset | UTF-8 字符编码，默认utf-8

```
a1.sources.r1.interceptors=i1
a1.sources.r1.interceptors.i1.type = search_replace
a1.sources.r1.interceptors.i1.searchPattern = [0-9]
a1.sources.r1.interceptors.i1.replaceString = *

Event:{headers:{ host=127.0.0.1,state=jp,addr=beijing,id=d354d2f0-14ff-4815-b382-99ae0a191926} body: 61 62 63 2A 2A 2A 79 2A 2A 2A 64 65 66  abc***y***def}
```

##### Regex Filtering Interceptor

此拦截器通过解析事件体去匹配正则表达式来筛选事件，所提供的正则表达式既可以用来包含或者刨除事件

配置项 | 说明
---|---
type (必填) | regex_filter
regex（必填）| ".*" 所要匹配的正则表达式
excludeEvents | 如果为true则刨除匹配事件，为false则包含匹配事件

```
# 结果将过滤以jp开头的信息，即如果发送的是以jp开头的信息，则收不到
a1.sources.r1.interceptors=i1
a1.sources.r1.interceptors.i1.type = regex_filter
a1.sources.r1.interceptors.i1.regex = ^jp.*$
a1.sources.r1.interceptors.i1.excludeEvents = true

测试：
[root@hadoop100 ~]# curl -X POST -d '[{"headers":{"state":"cn"},"body":"jpabc123y321def"}]' http://192.168.1.100:44444
[root@hadoop100 ~]# 
```

##### Regex Extractor Interceptor

使用指定正则表达式匹配事件，并将匹配的组作为头加入到事件中，它也支持插件化的序列化器用来格式化匹配到的组在加入他们作为头之前

配置项 | 说明
---|---
type (必填) | regex_extractor
regex（必填）| 要匹配的正则表达式
serializers（必填）|匹配对象列表

```
a1.sources.r1.interceptors=i1
a1.sources.r1.interceptors.i1.type = regex_extractor
a1.sources.r1.interceptors.i1.regex = (^[a-zA-Z]*)\\s([0-9]*$) # regex匹配并进行分组，匹配结果将有两部分，注意\s空白字符要进行转义
a1.sources.r1.interceptors.i1.serializers = s1 s2
a1.sources.r1.interceptors.i1.serializers.s1.name = word
a1.sources.r1.interceptors.i1.serializers.s2.name = digital

测试：
curl -X POST -d '[{"headers":{},"body":"zhangsan 1234"}]' http://192.168.1.100:44444

结果：
Event:{headers:{ word=zhangsan,digital=1234} body:73 68 61 6E 67 20 31 32 33 34    zhangsan 1234}
```
