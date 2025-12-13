# 第五章-14-Kafka集群部署[集群化软件]

Apache Kafka是一个分布式流处理平台，它提供高吞吐量、低延迟的实时数据处理能力。Kafka常用于构建实时数据管道、流式应用程序以及微服务架构中的消息中间件。本节将详细介绍如何在Linux集群环境中安装和部署Kafka集群。

## 1. 环境准备

*   **操作系统：** 至少三台Linux服务器（CentOS 7/8 或 Ubuntu Server 18.04/20.04），作为Kafka Broker节点。
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保所有节点之间可以互相访问，并根据**第五章-11-集群化软件安装前置准备**配置好主机名和SSH免密码登录。
*   **Java环境：** Kafka依赖于Java运行环境（JDK）。
*   **ZooKeeper集群：** Kafka依赖于ZooKeeper来存储元数据（如Broker、Topic、Partition等信息）。你需要先部署好一个ZooKeeper集群。参考**第五章-13-Zookeeper安装部署[集群化软件]**。

### 1.1 安装Java开发工具包 (JDK)

所有Kafka Broker节点都需要安装JDK。Kafka 2.x及以上版本通常建议使用JDK 8或JDK 11。

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

## 2. 下载Kafka

访问Apache Kafka官网（kafka.apache.org/downloads）获取最新稳定版的二进制包。选择Scala版本和对应的Kafka版本。

```bash
# 在一个节点上下载，然后通过scp分发到其他节点
wget https://downloads.apache.org/kafka/3.6.1/kafka_2.13-3.6.1.tgz # 请替换为最新版本和Scala版本
```

## 3. 安装Kafka (所有节点)

### 3.1 创建用户和安装目录

出于安全和权限管理考虑，建议创建专门的用户来运行Kafka。

```bash
sudo groupadd kafka
sudo useradd -s /bin/nologin -g kafka -d /opt/kafka kafka
sudo mkdir -p /opt/kafka
sudo mkdir -p /opt/kafka/data # 用于存储日志和数据
sudo chown -R kafka:kafka /opt/kafka
```

### 3.2 解压Kafka包

将下载的 `kafka_2.13-3.6.1.tgz` 移动到 `/opt/kafka` 目录下，并解压。

```bash
sudo mv kafka_2.13-3.6.1.tgz /opt/kafka/
cd /opt/kafka
sudo tar -zxvf kafka_2.13-3.6.1.tgz --strip-components=1
```
`--strip-components=1` 会去除压缩包内的顶层目录，直接将内容解压到 `/opt/kafka`。

### 3.3 分发Kafka到其他节点

使用 `scp` 命令将 `/opt/kafka` 目录复制到其他Broker节点。

```bash
# 在master节点执行
scp -r /opt/kafka/ kafka@slave1:/opt/
scp -r /opt/kafka/ kafka@slave2:/opt/
```
**注意：** 如果你在安装Kafka时没有使用专门的`kafka`用户，而是使用`root`用户，请确保在复制后修改文件的所有者和权限。

## 4. 配置Kafka集群 (所有节点)

Kafka的配置文件位于 `/opt/kafka/config/server.properties`。

### 4.1 修改 `server.properties` 配置文件

```bash
sudo vi /opt/kafka/config/server.properties
```

主要修改以下配置项：

*   **`broker.id`：** 每个Kafka Broker在集群中的唯一标识符。**每个节点必须不同，且为整数。**
    ```
    broker.id=0 # 节点1设置为0，节点2设置为1，以此类推
    ```
*   **`listeners`：** Kafka Broker监听的地址和端口。
    ```
    listeners=PLAINTEXT://<本机IP地址或主机名>:9092
    # 例如：
    listeners=PLAINTEXT://master-node:9092
    ```
    如果你希望其他机器通过主机名访问，确保主机名在 `/etc/hosts` 中已配置。
*   **`advertised.listeners`：** 供客户端连接的地址和端口。在单机或简单集群中可以与`listeners`相同。在NAT或Docker环境中可能需要设置为外部可访问的地址。
    ```
    advertised.listeners=PLAINTEXT://<本机IP地址或主机名>:9092
    # 例如：
    advertised.listeners=PLAINTEXT://master-node:9092
    ```
*   **`log.dirs`：** Kafka存储消息日志的路径。建议使用独立磁盘或分区。
    ```
    log.dirs=/opt/kafka/data/kafka-logs
    ```
    确保 `kafka` 用户对该目录有写权限。
*   **`num.partitions`：** 每个Topic默认创建的分区数。
    ```
    num.partitions=1
    ```
*   **`num.recovery.threads.per.data.dir`：** 每个数据目录用于恢复日志的线程数。
    ```
    num.recovery.threads.per.data.dir=1
    ```
*   **`offsets.topic.replication.factor`：** Offset Topic的副本因子。建议设置为集群节点数，以保证高可用。
    ```
    offsets.topic.replication.factor=3 # 如果有3个节点
    ```
*   **`transaction.state.log.replication.factor`：** 事务日志Topic的副本因子。
    ```
    transaction.state.log.replication.factor=3 # 如果有3个节点
    ```
*   **`transaction.state.log.min.isr`：** 事务日志Topic的最小ISR。
    ```
    transaction.state.log.min.isr=2
    ```
*   **`zookeeper.connect`：** ZooKeeper集群的连接字符串。
    ```
    zookeeper.connect=master-node:2181,slave1:2181,slave2:2181
    ```
    多个ZooKeeper地址用逗号分隔，`master-node`, `slave1`, `slave2` 是ZooKeeper节点的主机名。
*   **`zookeeper.connection.timeout.ms`：** ZooKeeper连接超时时间。
    ```
    zookeeper.connection.timeout.ms=6000
    ```

### 4.2 创建数据目录

在所有Broker节点上创建消息日志存储目录，并赋予`kafka`用户权限。
```bash
sudo mkdir -p /opt/kafka/data/kafka-logs
sudo chown -R kafka:kafka /opt/kafka/data/kafka-logs
```

## 5. 配置防火墙 (所有节点)

Kafka默认监听 `9092` 端口。所有Broker节点都需要开放此端口。

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-port=9092/tcp --permanent
sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 9092/tcp
sudo ufw enable           # 如果ufw未启用
sudo ufw status verbose
```

## 6. 启动Kafka集群 (所有节点)

在所有Broker节点上，使用Kafka自带的脚本启动服务。

```bash
# 切换到kafka用户
sudo su - kafka

# 启动Kafka Broker
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties
```
`-daemon` 参数会让Kafka在后台运行。

### 6.1 配置Systemd服务 (可选，推荐)

为了方便管理和开机自启，可以将Kafka配置为Systemd服务。

**创建Systemd服务文件（所有节点）：**
```bash
sudo vi /etc/systemd/system/kafka.service
```
将以下内容粘贴到文件中：

```ini
[Unit]
Description=Apache Kafka Server
Requires=zookeeper.service # 如果ZooKeeper和Kafka在同一台机器上，可以添加依赖
After=zookeeper.service network.target

[Service]
Type=forking
User=kafka
Group=kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
WorkingDirectory=/opt/kafka
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**重载Systemd并启动Kafka：**
```bash
sudo systemctl daemon-reload
sudo systemctl start kafka
sudo systemctl enable kafka
sudo systemctl status kafka
```

## 7. 验证Kafka集群

### 7.1 创建Topic

在任一Broker节点上创建一个Topic：
```bash
# 切换到kafka用户
sudo su - kafka

/opt/kafka/bin/kafka-topics.sh --create --topic my_test_topic --bootstrap-server master-node:9092,slave1:9092,slave2:9092 --partitions 3 --replication-factor 3
```
*   `--bootstrap-server`：指定任意一个或多个Broker的地址。
*   `--partitions 3`：创建3个分区。
*   `--replication-factor 3`：每个分区有3个副本。

### 7.2 列出Topic

```bash
/opt/kafka/bin/kafka-topics.sh --list --bootstrap-server master-node:9092
```

### 7.3 发送消息

在任一Broker节点上发送消息：
```bash
/opt/kafka/bin/kafka-console-producer.sh --broker-list master-node:9092 --topic my_test_topic
# 输入消息后回车发送
Hello Kafka
Another message
```

### 7.4 消费消息

在另一个终端或节点上消费消息：
```bash
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server master-node:9092 --topic my_test_topic --from-beginning
```
你应该能看到发送的消息。

## 8. 卸载Kafka

如果需要卸载Kafka：

```bash
sudo systemctl stop kafka         # 停止Kafka服务
sudo systemctl disable kafka      # 取消开机自启
sudo rm /etc/systemd/system/kafka.service # 删除Systemd服务文件
sudo systemctl daemon-reload      # 重载Systemd配置
sudo rm -rf /opt/kafka            # 删除Kafka安装目录
sudo userdel kafka                # 删除Kafka用户
sudo groupdel kafka               # 删除Kafka组
# 清理数据目录
sudo rm -rf /opt/kafka/data/kafka-logs
```

至此，你已成功在Linux集群环境中安装和部署了Apache Kafka，并了解了如何进行基本的管理和验证。
