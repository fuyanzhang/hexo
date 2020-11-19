---
title: docker中搭建spark集群
date: 2020-10-22 16:48:11
tags:
  - Docker
  - Spark
categories:
  - Spark
  - 技术
---

最近在做一些大数据分析的工作，之前没接触过，就使用 docker 搭建了一套 spark 集群，由于对 spark 了解不深入，这里就记录一下 docker 搭建 spark 集群的过程，备用。

<!--more-->

由于公司的大数据平台使用的 spark 比较老。这里使用跟公司相同的版本。
Scala: 2.10.4
Zookeeper: 3.4.9
Hadoop: 2.6.0
Spark: 1.5.2
JDK： openjdk1.8

## 环境及资源包准备

本文 docker 的宿主机系统为 CentOS7.docker 安装这里略过，不是本文的重点。
Scala 下载：https://scala-lang.org/files/archive/scala-2.10.4.tgz
Hadoop 下载：https://archive.apache.org/dist/hadoop/core/hadoop-2.6.0/hadoop-2.6.0.tar.gz
Zookeeper 下载：http://archive.apache.org/dist/zookeeper/zookeeper-3.4.9/zookeeper-3.4.9.tar.gz
Spark 下载： http://archive.apache.org/dist/spark/spark-1.5.2/spark-1.5.2-bin-hadoop2.6.tgz

### 准备基础镜像

1、使用 docker 拉取一个 Ubuntu 镜像`docker pull ubuntu`.
2、启动容器。`docker run -it ubuntu`
3、创建 spark 基础镜像。

> 1、在 Ubuntu 容器中创建文件夹/opt/soft.
> 2、将上面下载的资源包拷贝到容器中。通过命令`docker cp src_path containerId:dest_path`。其中 containerId 通过`docker ps`查看。
> 3、安装 jdk。进入容器，将 openjdk1.8 解压到/opt/soft/openjdk1.8.0 下。配置.bashrc 环境变量。

```
JAVA_HOME=/opt/soft/openjdk1.8.0
JRE_HOME=/opt/soft/openjdk1.8.0/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH
```

> 4、将 zookeeper 解压到/opt/soft/zookeeper/zookeeper-3.4.9 中。
> 5、配置 zoo.cfg，这里配置 3 个节点的集群。

```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just
# example sakes.
dataDir=/opt/soft/zookeeper/zookeeper-3.4.9/tmp
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
server.1=master:2888:3888
server.2=slave1:2888:3888
server.3=slave2:2888:3888
```

> 6、配置环境变量，修改.bashrc

```
export ZOOKEEPER_HOME=/opt/soft/zookeeper/zookeeper-3.4.9
export PATH=$PATH:$ZOOKEEPER_HOME/bin
```

> 7、在 tmp 目录下创建 myid 文件。
> 8、解压 Hadoop 到/opt/soft/hadoop 下
> 9、修改配置。
> 修改 slaves。

```
master
slave1
slave2
```

修改 hdfs-site.xml

```
<configuration>
        <property>
   <name>dfs.nameservices</name>
   <value>ns1</value>
</property>
<property>
   <name>dfs.ha.namenodes.ns1</name>
   <value>nn1,nn2</value>
</property>
<property>
   <name>dfs.namenode.rpc-address.ns1.nn1</name>
   <value>master:9000</value>
</property>
<property>
   <name>dfs.namenode.http-address.ns1.nn1</name>
   <value>master:50070</value>
</property>
<property>
   <name>dfs.namenode.rpc-address.ns1.nn2</name>
   <value>slave1:9000</value>
</property>
<property>
   <name>dfs.namenode.http-address.ns1.nn2</name>
   <value>slave1:50070</value>
</property>
<property>
   <name>dfs.namenode.shared.edits.dir</name>
<value>qjournal://master:8485;slave1:8485;slave2:8485/ns1</value>
</property>
<property>
   <name>dfs.journalnode.edits.dir</name>
   <value>/opt/soft/hadoop/hadoop-2.6.0/journal</value>
</property>
<property>
   <name>dfs.ha.automatic-failover.enabled</name>
   <value>true</value>
</property>
<property>
   <name>dfs.client.failover.proxy.provider.ns1</name>
   <value>
   org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider
   </value>
</property>
<property>
   <name>dfs.ha.fencing.methods</name>
   <value>
   sshfence
   shell(/bin/true)
   </value>
</property>
<property>
   <name>dfs.ha.fencing.ssh.private-key-files</name>
   <value>/root/.ssh/id_rsa</value>
</property>
<property>
   <name>dfs.ha.fencing.ssh.connect-timeout</name>
   <value>30000</value>
</property>
</configuration>
```

修改 core-site.xml

```
<configuration>
            <property>
         <name>hadoop.tmp.dir</name>
         <value>/opt/soft/hadoop/hadoop-2.6.0/tmp</value>
         <description>A base for other temporary directories.</description>
     </property>
     <property>
         <name>fs.default.name</name>
         <value>hdfs://master:9000</value>
         <final>true</final>
         <description>The name of the default file system.  A URI whose scheme and authority determine the FileSystem implementation.  The uri's scheme determines the config property (fs.SCHEME.impl) naming the FileSystem implementation class.  The uri's authority is used to determine the host, port, etc. for a filesystem.</description>
      </property>
</configuration>

```

修改 yarn-site.xml。

```
<property>
   <name>yarn.resourcemanager.hostname</name>
   <value>master</value>
</property>
<property>
   <name>yarn.nodemanager.aux-services</name>
   <value>mapreduce_shuffle</value>
</property>
</configuration>

```

修改 mapred-site.xml

```
<configuration>
        <property>
    <name>
      mapreduce.framework.name
    </name>
    <value>yarn</value>
</property>
</configuration>
```

> 10、配置环境变量。

```
export HADOOP_HOME=/opt/soft/hadoop/hadoop-2.6.0
export HADOOP_CONFIG_HOME=$HADOOP_HOME/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
```

> 11、安装 scala。解压 scala 包到/opt/soft/scala 下，配置环境变量。

```
export SCALA_HOME=/opt/soft/scala/scala-2.10.4
export PATH=$PATH:$SCALA_HOME/bin
```

> 12、安装 spark。解压 spark 到/opt/soft/spark 下。修改配置文件 spark-env.sh。

```
export SPARK_MASTER_IP=master
export SPARK_WORKER_MEMORY=2g
export JAVA_HOME=/opt/soft/openjdk1.8.0
export SCALA_HOME=/opt/soft/scala/scala-2.10.4
export SPARK_HOME=/opt/soft/spark/spark-1.5.2
export HADOOP_CONF_DIR=/opt/soft/hadoop/hadoop-2.6.0/etc/hadoop
export SPARK_LIBRARY_PATH=$SPARK_HOME/lib
export SCALA_LIBRARY_PATH=$SPARK_LIBRARY_PATH
export SPARK_WORKER_CORES=1
export SPARK_WORKER_INSTANCES=1
export SPARK_MASTER_PORT=7077

```

> 13、修改环境变量。

```
export SPARK_HOME=/opt/soft/spark/spark-1.5.2
export PATH=$SPARK_HOME/bin:$SPARK_HOME/sbin:$PATH
```

> 14、免密登录配置。由于 Hadoop 各个节点间需要免密登录，这里也将免密登录的信息放到镜像里。

```
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cd .ssh
cat id_rsa.pub >> authorized_keys
```

修改环境变量。

```
/usr/sbin/sshd
```

> 15、执行`source ~/.bashrc`，使配置的环境变量生效。
> **_到此，基础环境已经准备 ok 了，使用 docker commit -m "base spark" containerId ubuntu:spark 命令提交镜像_**
> 通过 docker run -it ubuntu:spark 可以进入镜像，执行各个模块的启动脚本可以启动了。
> 创建启动脚本：
> run_master.sh

```
#!/bin/bash
echo> /etc/hosts
echo 172.17.0.1 host >> /etc/hosts
echo 172.17.0.2 master >> /etc/hosts
echo 172.17.0.3 slave1 >> /etc/hosts
echo 172.17.0.4 slave2 >> /etc/hosts

echo 1 > /opt/soft/zookeeper/zookeeper-3.4.9/tmp/myid

zkServer.sh start &

hadoop-daemons.sh start journalnode
hdfs namenode -format
hdfs zkfc -formatZK

start-dfs.sh
start-yarn.sh
start-all.sh

```

run_slave1.sh

```
#!/bin/bash
echo> /etc/hosts
echo 172.17.0.1 host >> /etc/hosts
echo 172.17.0.2 master >> /etc/hosts
echo 172.17.0.3 slave1 >> /etc/hosts
echo 172.17.0.4 slave2 >> /etc/hosts

echo 2 > /opt/soft/zookeeper/zookeeper-3.4.9/tmp/myid

zkServer.sh start

```

run_slave2.sh

```
#!/bin/bash
echo> /etc/hosts
echo 172.17.0.1 host >> /etc/hosts
echo 172.17.0.2 master >> /etc/hosts
echo 172.17.0.3 slave1 >> /etc/hosts
echo 172.17.0.4 slave2 >> /etc/hosts

echo 3 > /opt/soft/zookeeper/zookeeper-3.4.9/tmp/myid

zkServer.sh start

```

stop_master.sh

```
#!/bin/bash
zkServer.sh stop
hadoop-daemons.sh stop journalnode
stop-dfs.sh
stop-yarn.sh
stop-all.sh
```

提交镜像。用 docker images 查看。
![spark集群镜像](/images/docker_spark_image.png)

### 集群启动

由于 docker 中的端口号不对外开放，所以在启动 master 容器时，需要跟宿主机端口做一个映射。命令为：
`docker run -it --privileged=true --name spark -h master -p 8081:8080 -p 2181:2181 -p 50070:50070 -p7077:7077 ubuntu:spark`
将 UI 相关的端口映射到宿主机上。（将 8080,2181,50070,7077 端口映射到宿主机的 8081,2181,50070,7077 上，由于我这边的宿主机端口 8080 被占用，所以这里映射到 8081 上）
启动 salve 容器，就不需要端口映射了。这里用`docker run -it --name slave1 -h slave1 ubuntu:spark`和`docker run -it --name slave2 -h slave2 ubuntu:spark`启动两个 slave 节点。

分别进入三个容器，执行对应的启动脚本`run_master.sh`,`run_slave1.sh`,`run_slave2.sh`。
在浏览器中访问`http://xx.xx.xx.xx:50070/dfshealth.html#tab-overview`，可以看到 Hadoop 相关信息。
![HadoopUI](/images/hadoop_ui.png)
在浏览器中访问`http://xx.xx.xx.xx:8081/`，可以看到 spark 相关信息.
![SparkUI](/images/spark_ui.png)

### 运行 spark 任务

编写一个 helloworld 程序。

```
object TestMain {

  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("myfirst")
    val sc = new SparkContext(conf)
    val rdd = sc.parallelize(List(1,2,3,4,5,6)).map(_*3)
    val mappedRDD = rdd.filter(_>10).collect()
    println(rdd.reduce(_+_))
    for (arg <- mappedRDD)
      println(arg+ " ")
    println()
    println("math is work")

  }
}
```

将上面代码打包成 jar 包。使用 spark-submit 命令将任务提交到集群上执行。
`./spark-submit --class com.study.spark.test.TestMain --master spark://xx.xx.xx.xx:7077 test.jar`
发现报了一个 akka 的错误，网上说 akka 只能通过 hostname 进行提交。
这里就需要修改 hostname。

```
172.17.0.1 host
172.17.0.2 master
172.17.0.3 slave1
172.17.0.4 slave2
```

再次执行命令，`./spark-submit --class com.study.spark.test.TestMain --master spark://master:7077 test.jar --driver-memory 32M --executor-memory 32M`。
任务成功运行。
![SparkUI](/images/spark任务运行结果.png)

### 参考

[从 0 开始使用 Docker 搭建 Spark 集群](https://www.jianshu.com/p/ee210190224f)
