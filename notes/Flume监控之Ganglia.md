#### 一、安装与部署

1. 安装httpd服务与php
```
[root@hadoop100 flume-1.6.0]# yum -y install httpd php
```
2. 安装其他依赖
```
[root@hadoop100 flume-1.6.0]# yum -y install rrdtool perl-rrdtool rrdtool-devel
[root@hadoop100 flume-1.6.0]# yum -y install apr-devel
```
3. 安装ganglia
```
//先要制作一个最简单的epel第三方yum安装配置:
[root@hadoop100 flume-1.6.0]# vim /etc/yum.repos.d/epel.repo 
[epel]
name=CentOS-$releasever - Epel
baseurl=http://dl.fedoraproject.org/pub/epel/$releasever/$basearch/
gpgcheck=0

[root@hadoop100 flume-1.6.0]# yum install libconfuse libconfuse-devel -y
[root@hadoop100 flume-1.6.0]# yum -y install ganglia-gmetad 
[root@hadoop100 flume-1.6.0]# yum -y install ganglia-web 
[root@hadoop100 flume-1.6.0]# yum -y install ganglia-gmond
```
4. 修改配置文件/etc/httpd/conf.d/ganglia.conf
```
[root@hadoop100 flume-1.6.0]# vim /etc/httpd/conf.d/ganglia.conf 
//注释掉其他的语句，添加Require all granted
<Location /ganglia>
  Require all granted
  # Require local
  # Require ip 10.1.2.3
  # Require host example.org
</Location>
```
5. 修改配置文件/etc/ganglia/gmetad.conf
```
//修改data_source
[root@hadoop100 flume-1.6.0]# vim /etc/ganglia/gmetad.conf 
data_source "hadoop100" 192.168.1.100
```
6. 修改配置文件/etc/ganglia/gmond.conf
```
[root@hadoop100 flume-1.6.0]# vim /etc/ganglia/gmond.conf 

//修改name
cluster {
  name = "hadoop100"
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}

//注释掉mcast_join,修改host
udp_send_channel {
  #bind_hostname = yes # Highly recommended, soon to be default.
                       # This option tells gmond to use a source address
                       # that resolves to the machine's hostname.  Without
                       # this, the metrics may appear to come from any
                       # interface and the DNS names associated with
                       # those IPs will be used to create the RRDs.
  # mcast_join = 239.2.11.71
  host = 192.168.1.100
  port = 8649
  ttl = 1
}

//注释掉mcast_join，添加bind
udp_recv_channel {
  # mcast_join = 239.2.11.71
  port = 8649
  bind = 192.168.1.100
  retry_bind = true
  # Size of the UDP buffer. If you are handling lots of metrics you really
  # should bump it up to e.g. 10MB or even higher.
  # buffer = 10485760
}
```
7. 修改配置文件/etc/selinux/config
```
[root@hadoop100 flume-1.6.0]# vim /etc/selinux/config
//修改SELINUX
SELINUX=disabled
```
8. 启动ganglia
```
[root@hadoop100 flume-1.6.0]# systemctl start httpd.service
[root@hadoop100 flume-1.6.0]# systemctl start gmetad.service
[root@hadoop100 flume-1.6.0]# systemctl start gmond.service
```
9. 打开网页浏览ganglia页面：http://192.168.1.100/ganglia

注意：如果完成以上操作依然出现权限不足错误，请修改/var/lib/ganglia目录的权限
```
[root@hadoop100 flume-1.6.0]# chmod -R 777 /var/lib/ganglia
```

#### 二、操作Flume测试监控

1. 修改/opt/module/flume/conf目录下的flume-env.sh配置
```
[root@hadoop100 flume-1.6.0]# vim /opt/module/flume-1.6.0/conf/flume-env.sh 
//添加如下配置
JAVA_OPTS="-Dflume.monitoring.type=ganglia
-Dflume.monitoring.hosts=192.168.1.100:8649
-Xms100m
-Xms200m"
```
2. 启动Flume任务
```
[root@hadoop100 job]# flume-ng agent \
-n a1 \
-f flume-nc-logger.conf \
-Dflume.root.logger==INFO,console \
-Dflume.monitoring.type=ganglia \
-Dflume.monitoring.hosts=192.168.1.100:8649
```

3. 发送数据观察ganglia监测图
```
[root@hadoop100 ~]# nc localhost 44444
```
图例说明：

字段（图标名称） | 字段含义
---|---
EventPutAttemptCount | source尝试写入channel的时间总数量
EventPutSuccessCount | 成功写入channel且提交的事件总数量
EventTakeAttemptCount | sink尝试从channel拉取事件的总数量，这不意味着每次事件都被返回，因为sink拉取的时候channel可能没有任何数据
EventTakeSuccessCount | sink成功读取的事件的总数量
StartTime | channel 启动的时间（毫秒）
StopTime | channel 停止的时间（毫秒）
ChannelSize | 目前channel中事件的总数量
ChannelFillPercentage | channel 占用百分比
ChannelCapacity | channel 的总容量

