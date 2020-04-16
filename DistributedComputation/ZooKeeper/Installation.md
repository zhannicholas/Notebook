> ZooKeeper是一个开源的分布式协调服务，可以为分布式计算环境维护配置信息、提供命名、同步、分组等服务。

# 单机部署

## 准备Java环境

ZooKeeper是由Java编写的，可以在`JDK 7`及以上版本中运行。在正式运行ZooKeeper之前，需要配置好Java环境以及`JAVA_HOME`。

## 下载

首先下载最版本的[ZooKeeper](http://zookeeper.apache.org/releases.html)，然后解压。写这篇文章的时候，最新的稳定版是3.5.5，所以我这里下载的也是这个版本。为了加快下载速度，我使用了清华大学的镜像站。
```shell
wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/stable/apache-zookeeper-3.5.5-bin.tar.gz
tar -xzvf apache-zookeeper-3.5.5-bin.tar.gz
```

## 创建配置文件

为了启动ZooKeeper，我们需要一份配置文件。启动时如果不指定配置文件名，ZooKeeper默认会去安装目录下的`conf`目录中寻找一个名为`zoo.cfg`的文件，并从中读取配置信息。先创建一份配置文件：
```shell
cd apache-zookeeper-3.5.5-bin
vim conf/zoo.cfg
```
下面是`zoo.cfg`文件内容的一个示例，文件内容参考自[ZooKeeper Getting Started Guide](http://zookeeper.apache.org/doc/r3.5.5/zookeeperStarted.html):
```
tickTime=2000
dataDir=/opt/Apache/apache-zookeeper-3.5.5-bin/data/standalone  # 数据目录
clientPort=2181
```
其实压缩包里附带了一份配置文件的样本`zoo_sample.cfg`，也可以参考这个文件进行修改。默认情况下，ZooKeeper内置的`AdminServer`使用了`8080`端口，如果安装机器上的`8080`端口已被使用，需要新增一个配置项，例如：
```
admin.serverPort=9090 # 使用9090端口作为AdminServer通信端口
```
还可以自定义ZooKeeper的日志文件夹，比如：
```
dataLogDir=/opt/Apache/apache-zookeeper-3.5.5-bin/logs/standalone
```

## 启动ZooKeeper
准备好配置文件之后，就可以启动ZooKeeper了:
```shell
bin/zkServer.sh start
```
若想显式指定使用配置文件`zoo.cfg`，则可运行：
```shell
bin/zkServer.sh start zoo.cfg
```

若看到出错信息，则ZooKeeper启动成功。可进一步查看当前运行状态：
```shell
bin/zkServer.sh status
```

## 连接ZooKeeper

```shell
bin/zkCli.sh -server 127.0.0.1:2181
```

## 停止ZooKeeper

```shell
bin/zkServer.sh stop
```

# 伪集群部署

很多时候，单机版的ZooKeeper已经能够满足我们的测试要求了。有时候，我们也需要模拟集群环境。ZooKeeper推荐在集群中使用**奇数**台机器。下面将部署的是3节点的伪集群。

## 准备配置文件

集群模式的配置文件在单机的基础上增加了一些和集群相关的信息，下面是一个例子，在`conf`目录中建立文件`zoo1.cfg`：
```
tickTime=2000                                                                                                          
dataDir=/opt/Apache/apache-zookeeper-3.5.5-bin/data/cluster/zk1
clientPort=2182
initLimit=10
syncLimit=5
dataLogDir=/opt/Apache/apache-zookeeper-3.5.5-bin/logs/cluster/zk1
server.1=127.0.0.1:2887:3887
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2889:3889                           
```
上面的`server.X`中，`X`代表服务器的id。服务器在启动时，会去读取`dataDir`目录下一个名为`myid`的文件。`myid`文件的内容即服务器的id。

同理，`zoo2.cfg`的内容为：
```
tickTime=2000                                                                                                          
dataDir=/opt/Apache/apache-zookeeper-3.5.5-bin/data/cluster/zk2
clientPort=2183
initLimit=10
syncLimit=5
dataLogDir=/opt/Apache/apache-zookeeper-3.5.5-bin/logs/cluster/zk2
server.1=127.0.0.1:2887:3887
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2889:3889   
```
`zoo3.cfg`:
```
tickTime=2000                                                                                                          
dataDir=/opt/Apache/apache-zookeeper-3.5.5-bin/data/cluster/zk3
clientPort=2184
initLimit=10
syncLimit=5
dataLogDir=/opt/Apache/apache-zookeeper-3.5.5-bin/logs/cluster/zk3
server.1=127.0.0.1:2887:3887
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2889:3889   
```
因为是伪集群，三个配置文件中的各个端口不一样，如果是真集群，端口可以是一样的。

## 准备myid文件

分别在上面三个配置文件指定的`dataDir`中建立一个名为`myid`的文件，文件内容为服务器的id：
```shell
echo "1" >> /opt/Apache/apache-zookeeper-3.5.5-bin/data/cluster/zk1/myid
echo "2" >> /opt/Apache/apache-zookeeper-3.5.5-bin/data/cluster/zk2/myid
echo "3" >> /opt/Apache/apache-zookeeper-3.5.5-bin/data/cluster/zk3/myid
```

## 启动伪集群

通过指定配置文件的方式启动各个节点：
```shell
bin/zkServer.sh start zoo1.cfg
bin/zkServer.sh start zoo2.cfg
bin/zkServer.sh start zoo3.cfg
```

查看各节点状态：
```shell
bin/zkServer.sh status zoo1.cfg
bin/zkServer.sh status zoo2.cfg
bin/zkServer.sh status zoo3.cfg
```

停止集群：
```shell
bin/zkServer.sh stop zoo1.cfg
bin/zkServer.sh stop zoo2.cfg
bin/zkServer.sh stop zoo3.cfg
```