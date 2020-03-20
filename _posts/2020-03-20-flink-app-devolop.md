---
title: Flink APP 开发及集群部署
category: flink
tags: flinka app deployment
---

* content
{:toc}

## Flink 简介

[Flink](https://flink.apache.org/)的简介请参考官网，不废话，直接上干货。

## 一、开发

### 1. 开发环境搭建（IDEA）

1. 下载[Flink源码](https://flink.apache.org/downloads.html)，并解压（假设解压到：/path/to/flink）。
2. 在[Idea中创建Java项目](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/projectsetup/java_api_quickstart.html)。
3. 项目创建好后，打开File->Project Structure->Libraries，添加/path/to/flink/lib和/path/to/flink/opt两个库。
4. 直接按普通Java程序运行或调试即可。




### 2. Flink应用中使用HDFS

- #### Maven依赖及对应Jar包

1. Maven依赖：
  
```xml
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-connector-filesystem_${scala.binary.version}</artifactId>
      <version>${flink.version}</version>
      <scope>provided</scope>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.flink/flink-shaded-hadoop2 -->
    <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-shaded-hadoop2</artifactId>
      <version>2.8.3-1.8.3</version>
    </dependency>
```

2. Jar包下载：

点击[这里](https://mvnrepository.com/artifact/org.apache.flink/flink-shaded-hadoop2)，选择对应版本后，点击下图所示位置下载flink-shaded-hadoop2的jar包，下载后放在/path/to/flink/lib/目录。

![下载flink-shaded-hadoop2][download_flink-shaded-hadoop2]

- #### 代码片段

```java
// Blink Planner
EnvironmentSettings envSettings =
    EnvironmentSettings.newInstance().useBlinkPlanner().inBatchMode().build();
final TableEnvironment tableEnv = TableEnvironment.create(envSettings);
......

// 创建并注册Table Sink
final Schema sinkSchema = new Schema()
    .field("1", DataTypes.STRING())
    .field("2", DataTypes.STRING())
    .field("3", DataTypes.STRING())
    .field("4", DataTypes.DECIMAL(38, 18))
    .field("5", DataTypes.DECIMAL(38, 18));
tableEnv.connect(newFileSystem().path("hdfs://ip:port/path/to/file"))
        .withFormat(new OldCsv())
        .withSchema(sinkSchema)
        .createTemporaryTable("CsvSinkTable");
```

- #### HDFS配置

必须指定Hadoop的配置文件位置才能在应用中正常使用HDFS，有两种方式（任选其一）：

1. 设定环境变量HADOOP_CONF_DIR（官方推荐方法）

```shell
HADOOP_CONF_DIR=/path/to/etc/hadoop
```

2. 在flink-conf.yaml中指定Hadoop配置文件位置：

```yml
fs.hdfs.hadoopconf: /path/to/etc/hadoop
```

这种方式适合在开发环境中使用，需把HDFS所在集群对应的etc/hadoop文件夹拷贝到开发机上，并在上述配置中指定即可。

## 二、部署

### 1. Local 模式部署（Windows）

1. 下载[Flink](https://flink.apache.org/downloads.html)，并解压（假设解压到：/path/to/flink）.
2. 进入/path/to/flink目录，修改配置文件：conf\flink-conf.yaml。
3. 使用如下命令启动本地集群:

```shell
D:\flink-1.10> .\bin\start-cluster.bat
```

- #### flink-conf.yaml

```yml
#==============================================================================
# Common
#==============================================================================

# The external address of the host on which the JobManager runs and can be
# reached by the TaskManagers and any clients which want to connect. This setting
# is only used in Standalone mode and may be overwritten on the JobManager side
# by specifying the --host <hostname> parameter of the bin/jobmanager.sh executable.
# In high availability mode, if you use the bin/start-cluster.sh script and setup
# the conf/masters file, this will be taken care of automatically. Yarn/Mesos
# automatically configure the host name based on the hostname of the node where the
# JobManager runs.

jobmanager.rpc.address: localhost

# The RPC port where the JobManager is reachable.

jobmanager.rpc.port: 6123

# The heap size for the JobManager JVM

jobmanager.heap.size: 1g

# The total process memory size for the TaskManager.
#
# Note this accounts for all memory usage within the TaskManager process, including JVM metaspace and other overhead.

#taskmanager.memory.process.size: 4g

# To exclude JVM metaspace and overhead, please, use total Flink memory size instead of 'taskmanager.memory.process.size'.
# It is not recommended to set both 'taskmanager.memory.process.size' and Flink memory.
#
taskmanager.memory.flink.size: 4g

# The number of task slots that each TaskManager offers. Each slot runs one parallel pipeline.

taskmanager.numberOfTaskSlots: 4

### TaskManager 启动报错，提示没有设置下列属性
taskmanager.cpu.cores: 2
taskmanager.memory.task.heap.size: 2g
taskmanager.memory.managed.size: 1g

# The parallelism used for programs that did not specify and other parallelism.

parallelism.default: 4

# The default file system scheme and authority.
# 
# By default file paths without scheme are interpreted relative to the local
# root file system 'file:///'. Use this to override the default and interpret
# relative paths relative to a different file system,
# for example 'hdfs://mynamenode:12345'
#
# fs.default-scheme

#==============================================================================
# High Availability
#==============================================================================

# The high-availability mode. Possible options are 'NONE' or 'zookeeper'.
#
high-availability: NONE

# The path where metadata for master recovery is persisted. While ZooKeeper stores
# the small ground truth for checkpoint and leader election, this location stores
# the larger objects, like persisted dataflow graphs.
# 
# Must be a durable file system that is accessible from all nodes
# (like HDFS, S3, Ceph, nfs, ...) 
#
# high-availability.storageDir: hdfs:///flink/ha/

# The list of ZooKeeper quorum peers that coordinate the high-availability
# setup. This must be a list of the form:
# "host1:clientPort,host2:clientPort,..." (default clientPort: 2181)
#
# high-availability.zookeeper.quorum: localhost:2181


# ACL options are based on https://zookeeper.apache.org/doc/r3.1.2/zookeeperProgrammers.html#sc_BuiltinACLSchemes
# It can be either "creator" (ZOO_CREATE_ALL_ACL) or "open" (ZOO_OPEN_ACL_UNSAFE)
# The default value is "open" and it can be changed to "creator" if ZK security is enabled
#
# high-availability.zookeeper.client.acl: open

#==============================================================================
# Fault tolerance and checkpointing
#==============================================================================

# The backend that will be used to store operator state checkpoints if
# checkpointing is enabled.
#
# Supported backends are 'jobmanager', 'filesystem', 'rocksdb', or the
# <class-name-of-factory>.
#
# state.backend: filesystem

# Directory for checkpoints filesystem, when using any of the default bundled
# state backends.
#
# state.checkpoints.dir: hdfs://namenode-host:port/flink-checkpoints

# Default target directory for savepoints, optional.
#
# state.savepoints.dir: hdfs://namenode-host:port/flink-checkpoints

# Flag to enable/disable incremental checkpoints for backends that
# support incremental checkpoints (like the RocksDB state backend). 
#
# state.backend.incremental: false

# The failover strategy, i.e., how the job computation recovers from task failures.
# Only restart tasks that may have been affected by the task failure, which typically includes
# downstream tasks and potentially upstream tasks if their produced data is no longer available for consumption.

jobmanager.execution.failover-strategy: region

#==============================================================================
# Rest & web frontend
#==============================================================================

# The port to which the REST client connects to. If rest.bind-port has
# not been specified, then the server will bind to this port as well.
#
rest.port: 8081

# The address to which the REST client will connect to
#
#rest.address: 0.0.0.0

# Port range for the REST and web server to bind to.
#
#rest.bind-port: 8080-8090

# The address that the REST & web server binds to
#
#rest.bind-address: 0.0.0.0

# Flag to specify whether job submission is enabled from the web-based
# runtime monitor. Uncomment to disable.

#web.submit.enable: false

#==============================================================================
# Advanced
#==============================================================================

# Override the directories for temporary files. If not specified, the
# system-specific Java temporary directory (java.io.tmpdir property) is taken.
#
# For framework setups on Yarn or Mesos, Flink will automatically pick up the
# containers' temp directories without any need for configuration.
#
# Add a delimited list for multiple directories, using the system directory
# delimiter (colon ':' on unix) or a comma, e.g.:
#     /data1/tmp:/data2/tmp:/data3/tmp
#
# Note: Each directory entry is read from and written to by a different I/O
# thread. You can include the same directory multiple times in order to create
# multiple I/O threads against that directory. This is for example relevant for
# high-throughput RAIDs.
#
# io.tmp.dirs: /tmp

# The classloading resolve order. Possible values are 'child-first' (Flink's default)
# and 'parent-first' (Java's default).
#
# Child first classloading allows users to use different dependency/library
# versions in their application than those in the classpath. Switching back
# to 'parent-first' may help with debugging dependency issues.
#
# classloader.resolve-order: child-first

# The amount of memory going to the network stack. These numbers usually need 
# no tuning. Adjusting them may be necessary in case of an "Insufficient number
# of network buffers" error. The default min is 64MB, the default max is 1GB.
# 
#taskmanager.memory.network.fraction: 0.1

### 这两个参数必须配置成相等，否则启动报错！
taskmanager.memory.network.min: 256mb
taskmanager.memory.network.max: 256mb

#==============================================================================
# Flink Cluster Security Configuration
#==============================================================================

# Kerberos authentication for various components - Hadoop, ZooKeeper, and connectors -
# may be enabled in four steps:
# 1. configure the local krb5.conf file
# 2. provide Kerberos credentials (either a keytab or a ticket cache w/ kinit)
# 3. make the credentials available to various JAAS login contexts
# 4. configure the connector to use JAAS/SASL

# The below configure how Kerberos credentials are provided. A keytab will be used instead of
# a ticket cache if the keytab path and principal are set.

# security.kerberos.login.use-ticket-cache: true
# security.kerberos.login.keytab: /path/to/kerberos/keytab
# security.kerberos.login.principal: flink-user

# The configuration below defines which JAAS login contexts

# security.kerberos.login.contexts: Client,KafkaClient

#==============================================================================
# ZK Security Configuration
#==============================================================================

# Below configurations are applicable if ZK ensemble is configured for security

# Override below configuration to provide custom ZK service name if configured
# zookeeper.sasl.service-name: zookeeper

# The configuration below must match one of the values set in "security.kerberos.login.contexts"
# zookeeper.sasl.login-context-name: Client

#==============================================================================
# HistoryServer
#==============================================================================

# The HistoryServer is started and stopped via bin/historyserver.sh (start|stop)

# Directory to upload completed jobs to. Add this directory to the list of
# monitored directories of the HistoryServer as well (see below).
#jobmanager.archive.fs.dir: hdfs:///completed-jobs/

# The address under which the web-based HistoryServer listens.
#historyserver.web.address: 0.0.0.0

# The port under which the web-based HistoryServer listens.
#historyserver.web.port: 8082

# Comma separated list of directories to monitor for completed jobs.
#historyserver.archive.fs.dir: hdfs:///completed-jobs/

# Interval in milliseconds for refreshing the monitored directories.
#historyserver.archive.fs.refresh-interval: 10000
```

### 2. Standalone 集群模式部署

1. 下载[Flink](https://flink.apache.org/downloads.html)，并解压（假设解压到：/path/to/flink）.
2. 进入/path/to/flink目录，修改配置文件：

   - conf/masters
   - conf/slaves
   - conf/flink-conf.yaml

3. 使用如下命令启动集群(会自动启动Masters和Slaves):

```shell
$ ./bin/start-cluster.sh
```

- #### masters

```
master:8081
```

- #### slaves（每个Slave一行）

```
slave1
slave2
slave3
```

- #### flink-conf.yaml

```yml
#==============================================================================
# Common
#==============================================================================

# The external address of the host on which the JobManager runs and can be
# reached by the TaskManagers and any clients which want to connect. This setting
# is only used in Standalone mode and may be overwritten on the JobManager side
# by specifying the --host <hostname> parameter of the bin/jobmanager.sh executable.
# In high availability mode, if you use the bin/start-cluster.sh script and setup
# the conf/masters file, this will be taken care of automatically. Yarn/Mesos
# automatically configure the host name based on the hostname of the node where the
# JobManager runs.

jobmanager.rpc.address: master

# The RPC port where the JobManager is reachable.

jobmanager.rpc.port: 6123


# The heap size for the JobManager JVM

jobmanager.heap.size: 2g


# The total process memory size for the TaskManager.
#
# Note this accounts for all memory usage within the TaskManager process, including JVM metaspace and other overhead.

#taskmanager.memory.process.size: 1568m

# To exclude JVM metaspace and overhead, please, use total Flink memory size instead of 'taskmanager.memory.process.size'.
# It is not recommended to set both 'taskmanager.memory.process.size' and Flink memory.
#
taskmanager.memory.flink.size: 4g

# The number of task slots that each TaskManager offers. Each slot runs one parallel pipeline.

taskmanager.numberOfTaskSlots: 4

### TaskManager 启动报错，提示没有设置下列属性
taskmanager.cpu.cores: 2
taskmanager.memory.task.heap.size: 2gb
taskmanager.memory.managed.size: 1gb

# The parallelism used for programs that did not specify and other parallelism.

parallelism.default: 12

# The default file system scheme and authority.
#
# By default file paths without scheme are interpreted relative to the local
# root file system 'file:///'. Use this to override the default and interpret
# relative paths relative to a different file system,
# for example 'hdfs://mynamenode:12345'
#
# fs.default-scheme

#==============================================================================
# High Availability
#==============================================================================

# The high-availability mode. Possible options are 'NONE' or 'zookeeper'.
#
high-availability: NONE

# The path where metadata for master recovery is persisted. While ZooKeeper stores
# the small ground truth for checkpoint and leader election, this location stores
# the larger objects, like persisted dataflow graphs.
#
# Must be a durable file system that is accessible from all nodes
# (like HDFS, S3, Ceph, nfs, ...)
#
# high-availability.storageDir: hdfs:///flink/ha/

# The list of ZooKeeper quorum peers that coordinate the high-availability
# setup. This must be a list of the form:
# "host1:clientPort,host2:clientPort,..." (default clientPort: 2181)
#
# high-availability.zookeeper.quorum: localhost:2181


# ACL options are based on https://zookeeper.apache.org/doc/r3.1.2/zookeeperProgrammers.html#sc_BuiltinACLSchemes
# It can be either "creator" (ZOO_CREATE_ALL_ACL) or "open" (ZOO_OPEN_ACL_UNSAFE)
# The default value is "open" and it can be changed to "creator" if ZK security is enabled
#
# high-availability.zookeeper.client.acl: open

#==============================================================================
# Fault tolerance and checkpointing
#==============================================================================

# The backend that will be used to store operator state checkpoints if
# checkpointing is enabled.
#
# Supported backends are 'jobmanager', 'filesystem', 'rocksdb', or the
# <class-name-of-factory>.
#
# state.backend: filesystem

# Directory for checkpoints filesystem, when using any of the default bundled
# state backends.
#
# state.checkpoints.dir: hdfs://namenode-host:port/flink-checkpoints

# Default target directory for savepoints, optional.
#
# state.savepoints.dir: hdfs://namenode-host:port/flink-checkpoints

# Flag to enable/disable incremental checkpoints for backends that
# support incremental checkpoints (like the RocksDB state backend).
#
# state.backend.incremental: false

# The failover strategy, i.e., how the job computation recovers from task failures.
# Only restart tasks that may have been affected by the task failure, which typically includes
# downstream tasks and potentially upstream tasks if their produced data is no longer available for consumption.

jobmanager.execution.failover-strategy: region

#==============================================================================
# Rest & web frontend
#==============================================================================

# The port to which the REST client connects to. If rest.bind-port has
# not been specified, then the server will bind to this port as well.
#
rest.port: 8081

# The address to which the REST client will connect to
#
rest.address: 0.0.0.0

# Port range for the REST and web server to bind to.
#
#rest.bind-port: 8080-8090

# The address that the REST & web server binds to
#
rest.bind-address: 0.0.0.0

# Flag to specify whether job submission is enabled from the web-based
# runtime monitor. Uncomment to disable.

#web.submit.enable: false

#==============================================================================
# Advanced
#==============================================================================

# Override the directories for temporary files. If not specified, the
# system-specific Java temporary directory (java.io.tmpdir property) is taken.
#
# For framework setups on Yarn or Mesos, Flink will automatically pick up the
# containers' temp directories without any need for configuration.
#
# Add a delimited list for multiple directories, using the system directory
# delimiter (colon ':' on unix) or a comma, e.g.:
#     /data1/tmp:/data2/tmp:/data3/tmp
#
# Note: Each directory entry is read from and written to by a different I/O
# thread. You can include the same directory multiple times in order to create
# multiple I/O threads against that directory. This is for example relevant for
# high-throughput RAIDs.
#
io.tmp.dirs: /data/tmp

# 上传JAR的存放位置
web.upload.dir: /opt/flink-1.10.0/jars

# The classloading resolve order. Possible values are 'child-first' (Flink's default)
# and 'parent-first' (Java's default).
#
# Child first classloading allows users to use different dependency/library
# versions in their application than those in the classpath. Switching back
# to 'parent-first' may help with debugging dependency issues.
#
# classloader.resolve-order: child-first

# The amount of memory going to the network stack. These numbers usually need
# no tuning. Adjusting them may be necessary in case of an "Insufficient number
# of network buffers" error. The default min is 64MB, the default max is 1GB.
#
# taskmanager.memory.network.fraction: 0.1
#taskmanager.memory.network.min: 256mb
#taskmanager.memory.network.max: 256mb

#==============================================================================
# Flink Cluster Security Configuration
#==============================================================================

# Kerberos authentication for various components - Hadoop, ZooKeeper, and connectors -
# may be enabled in four steps:
# 1. configure the local krb5.conf file
# 2. provide Kerberos credentials (either a keytab or a ticket cache w/ kinit)
# 3. make the credentials available to various JAAS login contexts
# 4. configure the connector to use JAAS/SASL

# The below configure how Kerberos credentials are provided. A keytab will be used instead of
# a ticket cache if the keytab path and principal are set.

# security.kerberos.login.use-ticket-cache: true
# security.kerberos.login.keytab: /path/to/kerberos/keytab
# security.kerberos.login.principal: flink-user

# The configuration below defines which JAAS login contexts

# security.kerberos.login.contexts: Client,KafkaClient

#==============================================================================
# ZK Security Configuration
#==============================================================================

# Below configurations are applicable if ZK ensemble is configured for security

# Override below configuration to provide custom ZK service name if configured
# zookeeper.sasl.service-name: zookeeper

# The configuration below must match one of the values set in "security.kerberos.login.contexts"
# zookeeper.sasl.login-context-name: Client

#==============================================================================
# HistoryServer
#==============================================================================

# The HistoryServer is started and stopped via bin/historyserver.sh (start|stop)

# Directory to upload completed jobs to. Add this directory to the list of
# monitored directories of the HistoryServer as well (see below).
#jobmanager.archive.fs.dir: hdfs:///completed-jobs/

# The address under which the web-based HistoryServer listens.
#historyserver.web.address: 0.0.0.0

# The port under which the web-based HistoryServer listens.
#historyserver.web.port: 8082

# Comma separated list of directories to monitor for completed jobs.
#historyserver.archive.fs.dir: hdfs:///completed-jobs/

# Interval in milliseconds for refreshing the monitored directories.
#historyserver.archive.fs.refresh-interval: 10000
```





[download_flink-shaded-hadoop2]:data:image/png;base64,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
