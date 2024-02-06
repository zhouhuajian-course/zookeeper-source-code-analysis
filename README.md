# Apache ZooKeeper 源码分析

## Windows 源码 一台机器 三个节点

1。 拷贝 zoo_sample.cfg 准备三个节点 配置
```
zoo1.cfg (另两个分别为zoo2 zoo3)

tickTime=2000
initLimit=10
syncLimit=5

dataDir=zkdata1 (另两个分别为zkdata2 zkdata3)
dataLogDir=zklog1 (另两个分别为zklog2 zklog3) (这个配置调试时可以不需要，默认为dataDir的值)
clientPort=2181 (另两个分别为2182 2183)

(不用 1 2 3是为了看日志方便)
server.101=localhost:2777:3777
server.102=localhost:2888:3888
server.103=localhost:2999:3999
```

2. 创建 myid
```
zkdata1/myid 里面 101 (不用 1 2 3是为了看日志方便)
zkdata2/myid 里面 102
zkdata3/myid 里面 103
```
3. 添加多三个 运行配置 ZooKeeper1 2 3，里面只需要修改参数为 conf/zoo1.cfg zoo2.cfg zoo3.cfg
4. 启动 ZooKeeper1 2 3
```
日志简要分析
ZooKeeper1 启动时
    Cannot open channel to 103 at election address localhost/127.0.0.1:3999
    Cannot open channel to 102 at election address localhost/127.0.0.1:3888
    (不停地尝试连接102 103 的竞选地址)

ZooKeeper2 启动时
    Sending UPTODATE message to 101
    Peer state changed: leading - broadcast (不会尝试连接103的竞选地址，已经选出leader)
    (ZooKeeper1 Peer state changed: following - broadcast 不再尝试连接103的竞选地址)

ZooKeeper 3 启动时
    Learner received UPTODATE message
    Peer state changed: following - broadcast
    (ZooKeeper 102 Sending UPTODATE message to 103)
    (ZooKeeper 101 There is a connection already for server 103)
```
5. 可视化工具

## 源码分析

IDEA 打开 apache-zookeeper-3.8.3

一般用 zkServer.sh 启动，可以在里面找到 启动类为 `org.apache.zookeeper.server.quorum.QuorumPeerMain`

配置 运行配置

![image_01.png](image/image_01.png)

运行前 compile 一下 

![image_02.png](image/image_02.png)

修改数据目录，很重要，方便后期调试

    conf/zoo_sample.cfg
    修改
    # dataDir=/tmp/zookeeper
    # 表示当前工作目录下，工作目录 运行配置 可修改，目录可以原来不存在
    dataDir=zookeeper_data

运行后 出现 Exception in thread "main" java.lang.NoClassDefFoundError: com/codahale/metrics/Reservoir

```
修改 zookeeper-server/pom.xml，注释掉 scope provided，然后 compile

    <dependency>
       <groupId>io.dropwizard.metrics</groupId>
       <artifactId>metrics-core</artifactId>
<!--       <scope>provided</scope>-->
    </dependency>
```

出现 java.lang.NoClassDefFoundError: org/xerial/snappy/SnappyInputStream

```
修改 zookeeper-server/pom.xml，注释掉 scope provided，然后 compile

    <dependency>
      <groupId>org.xerial.snappy</groupId>
      <artifactId>snappy-java</artifactId>
<!--      <scope>provided</scope>-->
    </dependency>
```

运行成功

## 可视化工具

例如

1. ZooKeeper Assistant
2. PrettyZoo (最后一次提交 3 weeks ago)
3. zktools
4. ZooInspector
5. zkui

等等

## 官网

https://zookeeper.apache.org/

## Basic

1. 类似文件系统+通知机制（类似监听者模式）
2. 每个节点都可以有数据和子节点，数据最多1M，ZK不适合做大数据存储，适合配置数据存储
3. 集群，只有一个leader，其他都是follower，半数以上的节点存活，集群才能用，所以一般使用 奇数节点部署
4. 不需要部署过多ZK，毕竟只是保存一些配置数据，例如20台Kafka Broker，7台ZK即可，能允许同时3台ZK宕机
5. 如果配置了集群，一开始启动ZK集群会选举一次Leader，根据Sid比较大，sid比较小的会把票给他，直到半数以上，后面启动的，因为有leader了，自动为follower
6. Leader宕机，会重新选择Leader，这时根据Epoch、Xid、Sid进行选举Leader
7. zkCli.sh help、get -s、create、delete、deleteall、ls -s 等等命令
8. get -w 可以监听节点数据变化，ls -w 可以监听节点下的路径变化
9. 注册一次监听器，只会监听一次，下次还想监听，需要重新注册
10. 默认有个zookeeper节点
11. 多个机器，ssh免密登录后，可以使用 case for ssh 命令结合，快速实现多台机器ZK的启动、停止和状态插叙，同样也可以实现多天机器jps快速查询
12. idea使用单元测试，可方便快速测试
13. 