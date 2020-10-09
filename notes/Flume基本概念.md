#### 一、概述
1. Flume最早是Cloudera提供的日志收集系统，后贡献给Apache
2. Flume是一个高可用的、高可靠的、分布式的海量日志采集、聚合和传输的系统
3. Flume支持在日志系统中定制各类数据发送方，用于收集数据（source）
4. Flume提供对数据的简单处理，并写到各种数据接收方（可定制）的能力（sink）

#### 二、版本
1. Flume0.9X：又称Flume-og，老版本的flume，需要引入zookeeper集群管理，性能也比较低（单线程工作）
2. Flume1.X：又称Flume-ng，新版本需要引入zookeeper，和Flume-og不兼容

#### 三、Flume特性
1. 可靠性：事务型的数据传递，保证数据的可靠性。一个日志交给flume来处理，不会出现此日志丢失或未被处理的及情况。
2. 可恢复性：通道可以以内存或文件的方式实现，内存更快，但不可恢复。文件较慢但提供了可恢复性。

#### 四、Flume组成架构

![](pics\flume_JG.jpg)

source不断的接收数据，将数据封装成一个一个的event，然后将event发送给channel，channel作为一个缓冲区会临时存放这些event数据，随后sink会将channel中的event数据发送到指定的地方，如hdfs等。当sink将channel中的数据成功发送出去之后，channel会将event数据进行删除，保证了数据传输的可靠性与安全性。

##### Agent
Flume以agent为最小的独立运行单位，一个agent就是一个JVM进程，主要负责将数据以事件的形式从源头送至目的端。它包含三个核心组件，分别是Source、Channel、Sink。

##### Source

主要负责收集数据，将接收到的数据封装到事件（event）中，然后将事件推送channel中。

在这个过程中它会先调用doPut()方法把批数据写入到临时缓冲区putList中，然后会检查channel中的容量是否足够，如果足够的话则会调用doCommit()方法把putList中的数据推送的channel中，如果channel容量不够的话，则会调用doRollBack()方法将数据回滚到putList中。

source可以处理的数据类型包括：netcat、exec、spooling directory、http、avro、thrift、jms、sequence generator、syslog、legacy等

##### Channel

是连接source和sink的组件，它可以将事件暂存到内存中也可以持久化到本地磁盘上，直到sink处理完该事件。它相当于一个数据的缓冲区。

flume自带两种channel：MemoryChannel、FileChannel
- MemoryChannel：将事件写入到内存中，可以实现高速的吞吐，但是无法保证数据的完整性，因为程序死亡、机器宕机或者重启都会导致数据丢失。

- FileChannel：将事件写到磁盘中，虽然效率相对较低，但是保证在程序关闭或者机器宕机的情况下不会丢失数据。

Channel是线程安全的，可以同时处理几个Source的写入操作和几个Sink的读取操作。

##### Sink

主要负责从channel中读取事件（event），将事件拉取到相应的存储系统，或者发送到另外一个Flume Agent中，并删除Channel中的缓存数据。

在这个过程中它会先调用 doTake() 方法把数据读取到临时缓冲区takeList中，然后检查数据是否发送成功，如果发送的话则调用 doCommit() 方法，把 event 从 takeList 中移除，如果发送失败，则调用 doRollBack() 方法把takeList中的数据回滚到 Channel 中。

Sink组件的目的地包括：hdfs、logger、avro、thrift、ipc、file、hbase、solr等

##### event

Flume数据传输的基本单元，以Event的形式将数据从源头送至目的地。它由Header和Body两部分组成，header是拦截器过滤好event之后，给event加的具体的header，用来存放该event的一些属性，为K-V结构，Body用来存放该条数据，形式为字节数组，所以一般都是拦截器和Multiplexing Channel Selector 结合起来使用。

#### 五. Flume拓扑结构

##### 单一流程

![](pics\DYLC.jpg)

##### 多个agent的数据流（多级流动）

![](pics\DDLLC.jpg)

##### 数据流合并（扇入流）

在做日志收集的时候一个常见的场景就是，大量的生产日志的客户端发送数据到少量的附属于存储子系统的消费者agent。例如，从数百个web服务器中收集日志，它们发送数据到十几个负责将数据写入HDFS集群的agent。

![](pics\LDHB.jpg)

这个在Flume中可以实现，需要配置大量第一层的agent，每一个agent都有一个avro sink,让它们都指向同一个agent的avro source（在这样的场景下也可以使用thrift source/sink/client）。在第二层agent上的source将收到的event合并到一个channel中，event被一个sink消费到它的最终目的地。

##### 数据流复用（扇出流）

Flume支持多路输出event流到一个或多个目的地。这是靠定义一个多路数据流实现的，它可以实现复制和选择性路由一个event到一个或者多个channel中。

![](pics\DLFY.jpg)

上面的例子展示了agent foo 中source 扇出数据流到三个不同的channel，这个扇出可以是复制或者多路输出。在复制数据流的情况下，每一个event被发送到所有的channel中；在多路输出的情况下，一个event被发送到一部分可用的channel中，它们是根据event的属性和预先配置的值选择channel的。这些映射关系应该被填写在agent的配置文件中。

##### 负载均衡功能

![](pics\FZJH.jpg)

#### 六、Interceptor拦截器

拦截器需要实现org.apache.flume.interceptor.Interceptor接口，通过拦截器可以实现在运行阶段修改/删除event。

拦截器采用了责任链模式，多个拦截器可以按指定顺序拦截，一个拦截器返回的事件列表被传递给链中的下一个拦截器。

如果一个拦截器需要删除事件，只需要在返回的事件集中不包含要删除的事件即可。如果要删除所有事件，只需要返回一个空列表。

#### 七、Channel Selectors

Channel选择器有两种类型：Replicating Channel Selector（默认的）和 Multiplexing Channel Selector。

- Replicating Channel Selector : 将source过来的events发往所有的channel（相当于复制多份）
- Multiplexing Channel Selector：可以根据event中header的值来配置具体发往哪一个Channel中。

#### 八、Flume Agent内部原理

![](pics\flume_agent_NBYL.jpg)

1. Source采集数据，将数据封装成事件对象(event)，然后交给Channel Processor
2. Channel Processor将事件传递给拦截器链进行简单的数据清洗   
3. 拦截器链处理完后将数据返回给Channel Processor。  
4. Channel Processor将拦截过滤之后的event事件传递给Channel选择器(Channel Selector)，由channel选择器决定每个event具体分配给哪一个Channel。     
5. Channel Selector返回给Channel Processor写入event事件的Channel列表。  
6. Channel Processor根据Channel选择器的选择结果，将Event事件写入相应的Channel中。  
7. 最后SinkProcessor启动sink，sink不断到channel中去轮询，将channel中的event事件拿过来。

#### 九、Flume安装

1. 将apache-flume-1.6.0-bin.tar.gz上传到Linux的/opt/software 目录下
2. 解压apache-flume-1.6.0-bin.tar.gz到/opt/module 目录下
```
[root@hadoop100 software]# tar -zxvf apache-flume-1.6.0-bin.tar.gz -C /opt/module/
```
3. 修改apache-flume-1.6.0-bin 的名称为flume-1.6.0
```
[root@hadoop100 module]# mv apache-flume-1.6.0-bin/ flume-1.6.0
```
4. 将flume/conf 下的flume-env.sh.template 文件修改为flume-env.sh，并配置flume-env.sh
```
[root@hadoop100 conf]# mv flume-env.sh.template flume-env.sh
[root@hadoop100 conf]# vim flume-env.sh
export JAVA_HOME=/opt/module/jdk1.8.0_40
```
5. 配置环境变量
```
[root@hadoop100 conf]# vim ~/.bashrc 
#FLUME
export FLUME_HOME=/opt/module/flume-1.6.0
export PATH=$PATH:$FLUME_HOME/bin
```
&emsp;&emsp;保存使其立即生效
```
[root@hadoop100 conf]# source ~/.bashrc 
```
6. 查看Flume版本
```
[root@hadoop100 conf]# flume-ng version
Flume 1.6.0
Source code repository: https://git-wip-us.apache.org/repos/asf/flume.git
Revision: 2561a23240a71ba20bf288c7c2cda88f443c2080
Compiled by hshreedharan on Mon May 11 11:15:44 PDT 2015
From source with checksum b29e416802ce9ece3269d34233baf43f
```
