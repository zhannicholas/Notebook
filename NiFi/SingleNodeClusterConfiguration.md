> 近段时间，工作中需要用到集群环境下的[Apache NiFi](https://nifi.apache.org/)。由于机器性能问题，笔者在自己的笔记本上搭建了只有一个节点的伪集群，用于测试。搭建过程中使用了外置的ZooKeeper。现在分享一下搭建过程，同时备忘。

# 系统环境及软件版本

* Linux Mint Debian Edition 3
* Java 1.8.0_221
* Apache NiFi 1.10.0
* Apache ZooKeeper 3.5.5

# 非安全模式（使用HTTP）下的安装过程

## 安装Java

为了运行ZooKeeper和NiFi，需要安装先安装Java。网上关于Java的安装教程数不胜数，这里不再赘述。安装完后需要设置`JAVA_HOME`。

## 安装ZooKeeper

这回使用了运行在standalone模式的ZooKeeper, 端口2181。

## 下载并安装NiFi

NiFi的安装包比较大，国内可以到清华大学开源软件镜像站下载。安装方法可以参考[Downloading and Installing NiFi](http://nifi.apache.org/docs/nifi-docs/html/getting-started.html#downloading-and-installing-nifi)。当本地可以成功启动NiFi之后，停止NiFi。

## 步骤

1. 进入NiFi安装目录下的`conf`目录，打开`nifi.properties`，进行编辑，改动的内容如下：

```properties
# web properties #
nifi.web.war.directory=./lib
# HTTP host, 默认为空白
nifi.web.http.host=127.0.0.1
# HTTP端口，默认8080
nifi.web.http.port=8080

# cluster node properties (only configure for cluster nodes) #
# 若为集群中的节点，这个值必须为true
nifi.cluster.is.node=true
# 节点的完全限定地址
nifi.cluster.node.address=127.0.0.1
# 节点的协议端口，默认为空白
nifi.cluster.node.protocol.port=8090
nifi.cluster.node.protocol.threads=10
nifi.cluster.node.protocol.max.threads=50
nifi.cluster.node.event.history.size=25
nifi.cluster.node.connection.timeout=5 sec
nifi.cluster.node.read.timeout=5 sec
nifi.cluster.node.max.concurrent.requests=100
nifi.cluster.firewall.file=
# 在选举出'正确'的流之前的可等待时间。若已投票的节点数量达到了nifi.cluster.flow.election.max.candidates的值，选举就结束。
# 默认的为5 min，我们只有一个节点，设置几秒钟就够了
nifi.cluster.flow.election.max.wait.time=10 sec
# 集群进行早期流选举所需的节点数量。因为我们只有一个节点，直接设置为1
nifi.cluster.flow.election.max.candidates=1

# zookeeper properties, used for cluster management #
# ZooKeeper连接串
nifi.zookeeper.connect.string=127.0.0.1:2181
nifi.zookeeper.connect.timeout=3 secs
nifi.zookeeper.session.timeout=3 secs
nifi.zookeeper.root.node=/nifi
```
2. 编辑`state-management.xml`，内容如下：
```xml
<cluster-provider>
    <id>zk-provider</id>
    <class>org.apache.nifi.controller.state.providers.zookeeper.ZooKeeperStateProvider</class>
    <property name="Connect String">127.0.0.1:2181</property>
    <property name="Root Node">/nifi</property>
    <property name="Session Timeout">10 seconds</property>
    <property name="Access Control">Open</property>
</cluster-provider>
```
注意这里的`Connect String`和`Root Node`的值要和`nifi.properties`中保持一致。

3. 启动NiFi, 访问http://127.0.0.1:8080 即可。如果不是在本机上访问，需要将配置中的`127.0.0.1`替换为本机IP.

# 安全模式（使用HTTPS）下的安装过程

要完成安全模式下集群的安装，可以先安装好运行在安全模式下的NiFi单机实例。

## 安装运行在安全模式的NiFi单机实例

为了让局域网内的其它机器也能访问安装好的NiFi。这里不再使用`127.0.0.1`作为配置文件中的IP，而是使用域名的方式。

### 修改host文件

在`/etc/hosts`中加入一条记录：
```
172.18.30.50 nifi-test.com
```
同时在其它想通过域名方式访问NiFi的机器的host文件里面也加上这一条记录。
使用域名进行配置的好处是：当主机的IP变化之后不需要重新修改配置文件，只用修改host文件就好了。

### 生成证书

[NiFi Toolkit](http://nifi.apache.org/docs/nifi-docs/html/toolkit-guide.html)自带了证书生成工具[TLS Toolkit](http://nifi.apache.org/docs/nifi-docs/html/toolkit-guide.html#tls_toolkit)，我们使用它证书。
NiFi的文档对证书生成有很详细的介绍，以下命令将为nifi-test.com生成证书，并放到`./target`文件夹下：
```
$ ./tls-toolkit.sh standalone -n 'nifi-test.com' -C 'CN=admin, OU=ApacheNiFi' -o './target'
```
命令执行完之后，`./target`文件夹的内容如下：
```txt
target/
|---- CN=admin_OU=ApacheNiFi.p12
|---- CN=admin_OU=ApacheNiFi.password
|---- nifi-cert.pem
|---- nifi-key.key
|---- nifi-test.com
      |---- keystore.jks
      |---- nifi.properties
      |---- truststore.jks
```

现在把`nifi-cert.pem`, `nifi-key.key`, `keystore.jks`, `nifi.properties`和`truststore.jks`