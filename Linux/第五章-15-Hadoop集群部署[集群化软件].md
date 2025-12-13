# 第五章-15-Hadoop集群部署[集群化软件]

Apache Hadoop是一个开源框架，用于存储和处理跨计算机集群的大型数据集。它提供了分布式文件系统HDFS（Hadoop Distributed File System）用于数据存储，以及MapReduce编程模型用于数据处理。Hadoop是大数据领域的基石。本节将详细介绍如何在Linux集群环境中部署Hadoop 3.x（伪分布式或完全分布式）。

## 1. 环境准备

*   **操作系统：** 至少两台Linux服务器（CentOS 7/8 或 Ubuntu Server 18.04/20.04），作为Hadoop集群的节点。一台作为主节点（NameNode、ResourceManager），其余作为从节点（DataNode、NodeManager）。
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保所有节点之间可以互相访问，并根据**第五章-11-集群化软件安装前置准备**配置好主机名和SSH免密码登录。
*   **Java环境：** Hadoop依赖于Java运行环境（JDK）。Hadoop 3.x通常需要JDK 8或JDK 11。

### 1.1 安装Java开发工具包 (JDK)

所有Hadoop节点都需要安装JDK。

**对于CentOS/RHEL (JDK 8为例)：**
```bash
sudo yum install java-1.8.0-openjdk-devel -y
```

**对于Ubuntu/Debian (JDK 8为例)：**
```bash
sudo apt update
sudo apt install openjdk-8-jdk -y
```

**配置JAVA_HOME环境变量（在所有节点上）：**
在 `/etc/profile` 或 `~/.bashrc` 中添加（根据实际路径修改）：
```bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64 # CentOS可能是 /usr/lib/jvm/java-1.8.0-openjdk
export PATH=$PATH:$JAVA_HOME/bin
```
然后 `source /etc/profile` 使其生效。

## 2. 下载Hadoop

访问Apache Hadoop官网（hadoop.apache.org/releases.html）获取最新稳定版的二进制包。

```bash
# 在一个节点上下载，然后通过scp分发到其他节点
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz # 请替换为最新版本
```

## 3. 安装Hadoop (所有节点)

### 3.1 创建用户和安装目录

出于安全和权限管理考虑，建议创建专门的用户来运行Hadoop。

```bash
sudo groupadd hadoop
sudo useradd -s /bin/bash -g hadoop -d /opt/hadoop hadoop
sudo mkdir -p /opt/hadoop/tmp     # HDFS临时目录
sudo mkdir -p /opt/hadoop/dfs/name # NameNode数据目录
sudo mkdir -p /opt/hadoop/dfs/data # DataNode数据目录
sudo mkdir -p /opt/hadoop/yarn/local # NodeManager本地数据目录
sudo mkdir -p /opt/hadoop/yarn/logs # NodeManager日志目录
sudo chown -R hadoop:hadoop /opt/hadoop
```
**注意：** 如果`hadoop`用户使用`bash`作为shell，将其`-s /bin/bash`设置为`bash`。

### 3.2 解压Hadoop包

将下载的 `hadoop-3.3.6.tar.gz` 移动到 `/opt/hadoop` 目录下，并解压。

```bash
sudo mv hadoop-3.3.6.tar.gz /opt/hadoop/
cd /opt/hadoop
sudo tar -zxvf hadoop-3.3.6.tar.gz --strip-components=1
```
`--strip-components=1` 会去除压缩包内的顶层目录，直接将内容解压到 `/opt/hadoop`。

### 3.3 分发Hadoop到其他节点

使用 `scp` 命令将 `/opt/hadoop` 目录复制到所有从节点。

```bash
# 在master节点执行
scp -r /opt/hadoop/ hadoop@slave1:/opt/
scp -r /opt/hadoop/ hadoop@slave2:/opt/
```

## 4. 配置Hadoop集群 (所有节点)

Hadoop的配置文件主要位于 `/opt/hadoop/etc/hadoop/` 目录下。

### 4.1 修改 `hadoop-env.sh` (所有节点)

编辑 `/opt/hadoop/etc/hadoop/hadoop-env.sh`：
```bash
sudo vi /opt/hadoop/etc/hadoop/hadoop-env.sh
```
找到 `export JAVA_HOME=` 这一行，将其修改为实际的JDK路径。
```bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64 # CentOS可能是 /usr/lib/jvm/java-1.8.0-openjdk
```

### 4.2 修改 `core-site.xml` (所有节点)

编辑 `/opt/hadoop/etc/hadoop/core-site.xml`：
```bash
sudo vi /opt/hadoop/etc/hadoop/core-site.xml
```
```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master-node:9000</value>
        <description>NameNode URI</description>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/hadoop/tmp</value>
        <description>A base for other temporary directories.</description>
    </property>
</configuration>
```

### 4.3 修改 `hdfs-site.xml` (所有节点)

编辑 `/opt/hadoop/etc/hadoop/hdfs-site.xml`：
```bash
sudo vi /opt/hadoop/etc/hadoop/hdfs-site.xml
```
```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value> <!-- 数据副本数，集群节点数大于等于此值 -->
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:///opt/hadoop/dfs/name</value>
        <description>Path on the local filesystem where the NameNode stores the namespace and transactional logs.</description>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:///opt/hadoop/dfs/data</value>
        <description>Comma separated list of paths on the local filesystem of a DataNode where it should store its blocks.</description>
    </property>
    <property>
        <name>dfs.namenode.http-address</name>
        <value>master-node:9870</value> <!-- NameNode Web UI端口 -->
    </property>
</configuration>
```

### 4.4 修改 `mapred-site.xml` (所有节点)

先从模板复制：
```bash
sudo cp /opt/hadoop/etc/hadoop/mapred-site.xml.template /opt/hadoop/etc/hadoop/mapred-site.xml
sudo vi /opt/hadoop/etc/hadoop/mapred-site.xml
```
```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
        <description>The runtime framework for executing MapReduce jobs.</description>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.map.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.reduce.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
</configuration>
```

### 4.5 修改 `yarn-site.xml` (所有节点)

编辑 `/opt/hadoop/etc/hadoop/yarn-site.xml`：
```bash
sudo vi /opt/hadoop/etc/hadoop/yarn-site.xml
```
```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>master-node</value> <!-- ResourceManager所在的主机名 -->
    </property>
    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>4096</value> <!-- NodeManager可用的最大内存 (MB) -->
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>512</value> <!-- 容器最小内存分配 (MB) -->
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>4096</value> <!-- 容器最大内存分配 (MB) -->
    </property>
    <property>
        <name>yarn.nodemanager.vcores</name>
        <value>2</value> <!-- NodeManager可用的最大CPU核心数 -->
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-vcores</name>
        <value>1</value> <!-- 容器最小CPU核心分配 -->
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-vcores</name>
        <value>2</value> <!-- 容器最大CPU核心分配 -->
    </property>
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value> <!-- 开启日志聚合 -->
    </property>
    <property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>604800</value> <!-- 日志聚合保留时间 (7天) -->
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>master-node:8088</value> <!-- ResourceManager Web UI端口 -->
    </property>
</configuration>
```

### 4.6 配置 `workers` 文件 (仅主节点)

编辑 `/opt/hadoop/etc/hadoop/workers`，列出所有从节点的主机名。
```bash
sudo vi /opt/hadoop/etc/hadoop/workers
```
```
slave1
slave2
```

## 5. 配置Hadoop环境变量 (所有节点)

在 `/etc/profile` 或 `~/.bashrc` 中添加Hadoop相关的环境变量：
```bash
export HADOOP_HOME=/opt/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HDFS_NAMENODE_USER=hadoop # 指定运行NameNode的用户
export HDFS_DATANODE_USER=hadoop # 指定运行DataNode的用户
export HDFS_SECONDARYNAMENODE_USER=hadoop # 指定运行SecondaryNameNode的用户
export YARN_RESOURCEMANAGER_USER=hadoop # 指定运行ResourceManager的用户
export YARN_NODEMANAGER_USER=hadoop # 指定运行NodeManager的用户
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
然后 `source /etc/profile` 使其生效。

## 6. 格式化HDFS (仅主节点，首次部署执行一次)

**重要：首次部署或重建集群时才执行此操作。此操作会删除HDFS上的所有数据。**
```bash
# 切换到hadoop用户
sudo su - hadoop
hdfs namenode -format
```
成功格式化后，会看到类似 `successfully formatted` 的信息。

## 7. 启动Hadoop集群

### 7.1 启动HDFS (仅主节点)

```bash
# 切换到hadoop用户
sudo su - hadoop
start-dfs.sh
```
这会启动NameNode和所有从节点上的DataNode。

### 7.2 启动YARN (仅主节点)

```bash
# 切换到hadoop用户
sudo su - hadoop
start-yarn.sh
```
这会启动ResourceManager和所有从节点上的NodeManager。

### 7.3 验证进程

在主节点上执行 `jps` 命令，应该看到 NameNode, ResourceManager, SecondaryNameNode (如果配置) 进程。
在从节点上执行 `jps` 命令，应该看到 DataNode, NodeManager 进程。

### 7.4 访问Web UI

*   **HDFS NameNode UI：** `http://master-node:9870`
*   **YARN ResourceManager UI：** `http://master-node:8088`

## 8. 配置防火墙 (所有节点)

需要开放Hadoop集群使用的端口。

**主要端口：**
*   `9000`: HDFS NameNode RPC端口 (core-site.xml: fs.defaultFS)
*   `9870`: HDFS NameNode Web UI端口 (hdfs-site.xml: dfs.namenode.http-address)
*   `9866`: HDFS DataNode Web UI端口 (hdfs-site.xml: dfs.datanode.http.address)
*   `8088`: YARN ResourceManager Web UI端口 (yarn-site.xml: yarn.resourcemanager.webapp.address)
*   `8042`: YARN NodeManager Web UI端口 (yarn-site.xml: yarn.nodemanager.webapp.address)
*   `19888`: MapReduce JobHistoryServer Web UI端口 (mapred-site.xml: mapreduce.jobhistory.webapp.address)

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-port=9000/tcp --permanent
sudo firewall-cmd --zone=public --add-port=9870/tcp --permanent
sudo firewall-cmd --zone=public --add-port=9866/tcp --permanent
sudo firewall-cmd --zone=public --add-port=8088/tcp --permanent
sudo firewall-cmd --zone=public --add-port=8042/tcp --permanent
sudo firewall-cmd --zone=public --add-port=19888/tcp --permanent
sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 9000/tcp
sudo ufw allow 9870/tcp
sudo ufw allow 9866/tcp
sudo ufw allow 8088/tcp
sudo ufw allow 8042/tcp
sudo ufw allow 19888/tcp
sudo ufw enable           # 如果ufw未启用
sudo ufw status verbose
```

## 9. 运行示例Hadoop作业

### 9.1 创建HDFS目录

```bash
# 切换到hadoop用户
sudo su - hadoop
hdfs dfs -mkdir /user
hdfs dfs -mkdir /user/hadoop
```

### 9.2 上传文件到HDFS

```bash
echo "Hello Hadoop" > input.txt
hdfs dfs -put input.txt /user/hadoop/input/
hdfs dfs -ls /user/hadoop/input/
```

### 9.3 运行WordCount示例

```bash
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar wordcount /user/hadoop/input /user/hadoop/output
```
等待作业完成。

### 9.4 查看结果

```bash
hdfs dfs -cat /user/hadoop/output/part-r-00000
```

## 10. 停止Hadoop集群

### 10.1 停止YARN (仅主节点)

```bash
# 切换到hadoop用户
sudo su - hadoop
stop-yarn.sh
```

### 10.2 停止HDFS (仅主节点)

```bash
# 切换到hadoop用户
sudo su - hadoop
stop-dfs.sh
```

## 11. 卸载Hadoop

如果需要卸载Hadoop：

```bash
sudo systemctl stop hadoop* # 如果配置了Systemd服务
sudo rm -rf /opt/hadoop            # 删除Hadoop安装目录
sudo rm -rf /opt/hadoop/tmp        # 删除HDFS临时目录
sudo rm -rf /opt/hadoop/dfs/name   # 删除NameNode数据目录
sudo rm -rf /opt/hadoop/dfs/data   # 删除DataNode数据目录
sudo rm -rf /opt/hadoop/yarn/local # 删除NodeManager本地数据目录
sudo rm -rf /opt/hadoop/yarn/logs  # 删除NodeManager日志目录
sudo userdel hadoop                # 删除Hadoop用户
sudo groupdel hadoop               # 删除Hadoop组
# 清理环境变量 (从 /etc/profile 或 ~/.bashrc 中删除)
```

至此，你已成功在Linux集群环境中安装和部署了Apache Hadoop，并了解了如何进行基本的管理和测试。
