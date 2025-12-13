# 第五章-18-Flink分布式内存计算集群部署[集群化软件]

Apache Flink是一个开源的流处理框架，用于有界和无界数据流的统一处理。它提供了高吞吐、低延迟的流式数据处理能力，并支持事件时间处理、状态管理、容错以及复杂事件处理（CEP）等功能。Flink可以运行在Standalone、YARN、Kubernetes等多种集群模式下。本节将详细介绍在Linux集群环境中部署Flink Standalone模式集群。

## 1. 环境准备

*   **操作系统：** 至少两台Linux服务器（CentOS 7/8 或 Ubuntu Server 18.04/20.04），一台作为JobManager，其余作为TaskManager。
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保所有节点之间可以互相访问，并根据**第五章-11-集群化软件安装前置准备**配置好主机名和SSH免密码登录。
*   **Java环境：** Flink依赖于Java运行环境（JDK）。Flink 1.x版本通常需要JDK 8或JDK 11。

### 1.1 安装Java开发工具包 (JDK)

所有Flink节点都需要安装JDK。

**配置JAVA_HOME环境变量（在所有节点上）：**
```bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64 # 根据实际路径修改
export PATH=$PATH:$JAVA_HOME/bin
```
然后 `source /etc/profile` 使其生效。

## 2. 下载Flink

访问Apache Flink官网（flink.apache.org/downloads/）获取最新稳定版的二进制包。选择对应的Scala版本和Hadoop版本。

```bash
# 在一个节点上下载，然后通过scp分发到其他节点
wget https://archive.apache.org/dist/flink/flink-1.18.1/flink-1.18.1-bin-scala_2.12.tgz # 请替换为最新版本
```

## 3. 安装Flink (所有节点)

### 3.1 创建用户和安装目录

出于安全和权限管理考虑，建议创建专门的用户来运行Flink。

```bash
sudo groupadd flink
sudo useradd -s /bin/bash -g flink -d /opt/flink flink
sudo mkdir -p /opt/flink
sudo chown -R flink:flink /opt/flink
```

### 3.2 解压Flink包

将下载的 `flink-1.18.1-bin-scala_2.12.tgz` 移动到 `/opt/flink` 目录下，并解压。

```bash
sudo mv flink-1.18.1-bin-scala_2.12.tgz /opt/flink/
cd /opt/flink
sudo tar -zxvf flink-1.18.1-bin-scala_2.12.tgz --strip-components=1
```
`--strip-components=1` 会去除压缩包内的顶层目录，直接将内容解压到 `/opt/flink`。

### 3.3 分发Flink到其他节点

使用 `scp` 命令将 `/opt/flink` 目录复制到所有TaskManager节点。

```bash
# 在jobmanager节点执行
scp -r /opt/flink/ flink@slave1:/opt/
scp -r /opt/flink/ flink@slave2:/opt/
```

## 4. 配置Flink集群 (所有节点)

Flink的配置文件主要位于 `/opt/flink/conf/` 目录下。

### 4.1 修改 `flink-conf.yaml` (所有节点)

编辑 `/opt/flink/conf/flink-conf.yaml`：
```bash
sudo vi /opt/flink/conf/flink-conf.yaml
```
主要修改以下配置项：

*   **`jobmanager.rpc.address`：** JobManager的主机名。
    ```yaml
    jobmanager.rpc.address: jobmanager-node
    ```
*   **`jobmanager.rpc.port`：** JobManager RPC端口，默认 `6123`。
    ```yaml
    jobmanager.rpc.port: 6123
    ```
*   **`jobmanager.memory.process.size`：** JobManager进程的总内存。
    ```yaml
    jobmanager.memory.process.size: 1600m
    ```
*   **`taskmanager.memory.process.size`：** TaskManager进程的总内存。
    ```yaml
    taskmanager.memory.process.size: 1728m
    ```
*   **`taskmanager.numberOfTaskSlots`：** 每个TaskManager可用的任务槽数。通常设置为与CPU核心数相同。
    ```yaml
    taskmanager.numberOfTaskSlots: 1
    ```
*   **`parallelism.default`：** 默认并行度。建议设置为所有TaskManager的任务槽总数。
    ```yaml
    parallelism.default: 3 # 如果有3个TaskManager，每个1个任务槽
    ```
*   **`rest.port`：** Flink Web UI端口。
    ```yaml
    rest.port: 8081
    ```
*   **`high-availability` (可选，推荐在生产环境配置)：** 配置高可用性。例如使用ZooKeeper：
    ```yaml
    high-availability: zookeeper
    high-availability.storageDir: hdfs://master-node:9000/flink/recovery # 存储JobManager元数据，需要HDFS
    high-availability.zookeeper.quorum: jobmanager-node:2181,slave1:2181,slave2:2181
    high-availability.zookeeper.path: /flink
    ```

### 4.2 修改 `masters` 文件 (仅JobManager节点)

编辑 `/opt/flink/conf/masters`，列出JobManager的主机名。
```bash
sudo vi /opt/flink/conf/masters
```
```
jobmanager-node
```

### 4.3 修改 `workers` 文件 (仅JobManager节点)

编辑 `/opt/flink/conf/workers`，列出所有TaskManager节点的主机名。
```bash
sudo vi /opt/flink/conf/workers
```
```
slave1
slave2
```

## 5. 配置Flink环境变量 (所有节点)

在 `/etc/profile` 或 `~/.bashrc` 中添加Flink相关的环境变量：
```bash
export FLINK_HOME=/opt/flink
export PATH=$PATH:$FLINK_HOME/bin
```
然后 `source /etc/profile` 使其生效。

## 6. 启动Flink集群 (仅JobManager节点)

**重要：如果配置了高可用性，确保HDFS和ZooKeeper集群已正常运行。**

```bash
# 切换到flink用户
sudo su - flink
start-cluster.sh
```
这会启动Flink JobManager和所有TaskManager。

### 6.1 验证进程

在JobManager节点上执行 `jps` 命令，应该看到 JobManager 进程。
在TaskManager节点上执行 `jps` 命令，应该看到 TaskManager 进程。

### 6.2 访问Web UI

*   **Flink Web UI：** `http://jobmanager-node:8081`

## 7. 配置防火墙 (所有节点)

需要开放Flink集群使用的端口。

**主要端口：**
*   `6123`: JobManager RPC端口
*   `8081`: Flink Web UI端口 (默认)
*   TaskManager数据通信端口：通常在JobManager和TaskManager启动时动态分配，但可以通过配置限制范围。

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-port=6123/tcp --permanent
sudo firewall-cmd --zone=public --add-port=8081/tcp --permanent
# 如果需要，开放TaskManager数据通信端口范围，例如：
# sudo firewall-cmd --zone=public --add-port=6121-6125/tcp --permanent
sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 6123/tcp
sudo ufw allow 8081/tcp
# 如果需要，开放TaskManager数据通信端口范围：
# sudo ufw allow 6121:6125/tcp
sudo ufw enable           # 如果ufw未启用
sudo ufw status verbose
```

## 8. 运行示例Flink作业

### 8.1 运行StreamWordCount

```bash
# 切换到flink用户
sudo su - flink
flink run -c org.apache.flink.streaming.examples.wordcount.WordCount \
  $FLINK_HOME/examples/streaming/StreamWordCount.jar
```
这会提交一个WordCount流处理作业。你可以通过Web UI查看作业状态。
如果你希望作业持续运行并监听端口输入，可以这样启动：
```bash
flink run -c org.apache.flink.streaming.examples.wordcount.SocketTextStreamWordCount \
  $FLINK_HOME/examples/streaming/StreamWordCount.jar \
  --port 9000
```
然后在另一个终端连接到 `jobmanager-node:9000` 并输入文本，即可看到WordCount结果。

## 9. 停止Flink集群 (仅JobManager节点)

```bash
# 切换到flink用户
sudo su - flink
stop-cluster.sh
```

## 10. 卸载Flink

如果需要卸载Flink：

```bash
sudo systemctl stop flink* # 如果配置了Systemd服务
sudo rm -rf /opt/flink            # 删除Flink安装目录
sudo userdel flink                # 删除Flink用户
sudo groupdel flink               # 删除Flink组
# 清理环境变量 (从 /etc/profile 或 ~/.bashrc 中删除)
```

至此，你已成功在Linux集群环境中安装和部署了Apache Flink Standalone模式，并了解了如何进行基本的管理和测试。
