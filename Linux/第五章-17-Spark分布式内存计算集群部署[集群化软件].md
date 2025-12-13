# 第五章-17-Spark分布式内存计算集群部署[集群化软件]

Apache Spark是一个强大的开源分布式计算系统，专注于大规模数据处理。它提供了比Hadoop MapReduce更快的处理速度、更丰富的API（支持SQL、流处理、机器学习、图计算）和更广泛的应用场景。Spark可以运行在Hadoop YARN、Apache Mesos或其自带的Standalone模式下。本节将详细介绍在Linux集群环境中部署Spark Standalone模式集群。

## 1. 环境准备

*   **操作系统：** 至少两台Linux服务器（CentOS 7/8 或 Ubuntu Server 18.04/20.04），一台作为Master，其余作为Worker。
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保所有节点之间可以互相访问，并根据**第五章-11-集群化软件安装前置准备**配置好主机名和SSH免密码登录。
*   **Java环境：** Spark依赖于Java运行环境（JDK）。Spark 3.x通常需要JDK 8或JDK 11。

### 1.1 安装Java开发工具包 (JDK)

所有Spark节点都需要安装JDK。

**配置JAVA_HOME环境变量（在所有节点上）：**
```bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64 # 根据实际路径修改
export PATH=$PATH:$JAVA_HOME/bin
```
然后 `source /etc/profile` 使其生效。

## 2. 下载Spark

访问Apache Spark官网（spark.apache.org/downloads.html）获取最新稳定版的预编译包。选择Hadoop版本和Spark版本。

```bash
# 在一个节点上下载，然后通过scp分发到其他节点
wget https://dlcdn.apache.org/spark/spark-3.5.0/spark-3.5.0-bin-hadoop3.tgz # 请替换为最新版本
```

## 3. 安装Spark (所有节点)

### 3.1 创建用户和安装目录

出于安全和权限管理考虑，建议创建专门的用户来运行Spark。

```bash
sudo groupadd spark
sudo useradd -s /bin/bash -g spark -d /opt/spark spark
sudo mkdir -p /opt/spark
sudo chown -R spark:spark /opt/spark
```

### 3.2 解压Spark包

将下载的 `spark-3.5.0-bin-hadoop3.tgz` 移动到 `/opt/spark` 目录下，并解压。

```bash
sudo mv spark-3.5.0-bin-hadoop3.tgz /opt/spark/
cd /opt/spark
sudo tar -zxvf spark-3.5.0-bin-hadoop3.tgz --strip-components=1
```
`--strip-components=1` 会去除压缩包内的顶层目录，直接将内容解压到 `/opt/spark`。

### 3.3 分发Spark到其他节点

使用 `scp` 命令将 `/opt/spark` 目录复制到所有Worker节点。

```bash
# 在master节点执行
scp -r /opt/spark/ spark@slave1:/opt/
scp -r /opt/spark/ spark@slave2:/opt/
```

## 4. 配置Spark集群 (所有节点)

Spark的配置文件主要位于 `/opt/spark/conf/` 目录下。

### 4.1 修改 `spark-env.sh` (所有节点)

先从模板复制：
```bash
sudo cp /opt/spark/conf/spark-env.sh.template /opt/spark/conf/spark-env.sh
sudo vi /opt/spark/conf/spark-env.sh
```
主要添加或修改以下配置项：

```bash
# Set Java Home
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64 # 根据实际路径修改

# Spark Master hostname
export SPARK_MASTER_HOST=master-node # Spark Master节点的主机名

# Set maximum memory for Spark Master and Worker
export SPARK_MASTER_WEBUI_PORT=8080 # Master Web UI端口
export SPARK_MASTER_PORT=7077 # Master RPC端口
export SPARK_WORKER_CORES=2 # 每个Worker节点使用的CPU核心数
export SPARK_WORKER_MEMORY=4g # 每个Worker节点使用的内存大小
export SPARK_WORKER_PORT=8081 # Worker RPC端口
export SPARK_WORKER_WEBUI_PORT=8081 # Worker Web UI端口

# Set the maximum number of cores for the Master (Optional)
# export SPARK_MASTER_CORES=4
```

### 4.2 配置 `workers` 文件 (仅Master节点)

先从模板复制：
```bash
sudo cp /opt/spark/conf/workers.template /opt/spark/conf/workers
sudo vi /opt/spark/conf/workers
```
列出所有Worker节点的主机名。
```
slave1
slave2
```

### 4.3 修改 `spark-defaults.conf` (所有节点, 可选)

先从模板复制：
```bash
sudo cp /opt/spark/conf/spark-defaults.conf.template /opt/spark/conf/spark-defaults.conf
sudo vi /opt/spark/conf/spark-defaults.conf
```
可以设置一些默认参数，例如：
```
spark.eventLog.enabled           true
spark.eventLog.dir               hdfs://master-node:9000/spark-logs # 如果有HDFS，可以存到HDFS
spark.history.fs.logDirectory    hdfs://master-node:9000/spark-logs
spark.master                     spark://master-node:7077
```

## 5. 配置Spark环境变量 (所有节点)

在 `/etc/profile` 或 `~/.bashrc` 中添加Spark相关的环境变量：
```bash
export SPARK_HOME=/opt/spark
export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin
```
然后 `source /etc/profile` 使其生效。

## 6. 启动Spark集群 (仅Master节点)

**重要：确保Hadoop HDFS（如果Spark数据存放在HDFS上）已正常运行。**

```bash
# 切换到spark用户
sudo su - spark
start-all.sh
```
这会启动Spark Master和所有Worker。

### 6.1 验证进程

在Master节点上执行 `jps` 命令，应该看到 Master 进程。
在Worker节点上执行 `jps` 命令，应该看到 Worker 进程。

### 6.2 访问Web UI

*   **Spark Master UI：** `http://master-node:8080`
*   **Spark Worker UI：** `http://slave1:8081`, `http://slave2:8081`

## 7. 配置防火墙 (所有节点)

需要开放Spark集群使用的端口。

**主要端口：**
*   `7077`: Spark Master RPC端口 (默认)
*   `8080`: Spark Master Web UI端口 (默认)
*   `8081`: Spark Worker Web UI端口 (默认)
*   `8081`: Spark Worker RPC端口 (默认)
*   `4040-4045`: Spark应用程序默认使用的端口范围。

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-port=7077/tcp --permanent
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --zone=public --add-port=8081/tcp --permanent
sudo firewall-cmd --zone=public --add-port=4040-4045/tcp --permanent
sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 7077/tcp
sudo ufw allow 8080/tcp
sudo ufw allow 8081/tcp
sudo ufw allow 4040:4045/tcp
sudo ufw enable           # 如果ufw未启用
sudo ufw status verbose
```

## 8. 运行示例Spark作业

### 8.1 运行Spark Shell

```bash
# 切换到spark用户
sudo su - spark
spark-shell --master spark://master-node:7077 --executor-memory 512m --num-executors 2
```
这会启动一个交互式的Spark Shell，连接到Master节点。

### 8.2 运行SparkPi示例

```bash
# 切换到spark用户
sudo su - spark
$SPARK_HOME/bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master spark://master-node:7077 \
  --deploy-mode cluster \
  --executor-memory 512m \
  --num-executors 2 \
  $SPARK_HOME/examples/jars/spark-examples_2.12-3.5.0.jar \
  10
```
这会计算Pi的值，并将作业提交到Spark集群。你可以通过Master UI查看作业状态。

## 9. 停止Spark集群 (仅Master节点)

```bash
# 切换到spark用户
sudo su - spark
stop-all.sh
```

## 10. 卸载Spark

如果需要卸载Spark：

```bash
sudo systemctl stop spark* # 如果配置了Systemd服务
sudo rm -rf /opt/spark            # 删除Spark安装目录
sudo userdel spark                # 删除Spark用户
sudo groupdel spark               # 删除Spark组
# 清理环境变量 (从 /etc/profile 或 ~/.bashrc 中删除)
```

至此，你已成功在Linux集群环境中安装和部署了Apache Spark Standalone模式，并了解了如何进行基本的管理和测试。
