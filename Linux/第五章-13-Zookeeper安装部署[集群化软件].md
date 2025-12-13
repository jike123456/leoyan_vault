# 第五章-13-Zookeeper安装部署[集群化软件]

Apache ZooKeeper是一个开源的分布式协调服务，用于管理大型分布式系统中的配置信息、提供命名服务、分布式同步以及组服务。它是Hadoop、Kafka、HBase等许多大型分布式系统的核心组件。本节将详细介绍如何在Linux集群环境中安装和部署ZooKeeper。

## 1. 环境准备

*   **操作系统：** 至少三台Linux服务器（CentOS 7/8 或 Ubuntu Server 18.04/20.04），作为ZooKeeper集群的节点。建议使用奇数个节点以避免脑裂。
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保所有节点之间可以互相访问，并根据**第五章-11-集群化软件安装前置准备**配置好主机名和SSH免密码登录。
*   **Java环境：** ZooKeeper依赖于Java运行环境（JDK）。

### 1.1 安装Java开发工具包 (JDK)

所有ZooKeeper节点都需要安装JDK。建议安装JDK 8。

**对于CentOS/RHEL：**
```bash
sudo yum install java-1.8.0-openjdk-devel -y
```

**对于Ubuntu/Debian：**
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

## 2. 下载ZooKeeper

访问Apache ZooKeeper官网（zookeeper.apache.org/releases.html）获取最新稳定版的二进制包。选择 `tar.gz` 格式。

```bash
# 在一个节点上下载，然后通过scp分发到其他节点
wget https://dlcdn.apache.org/zookeeper/zookeeper-3.8.3/apache-zookeeper-3.8.3-bin.tar.gz
```

## 3. 安装ZooKeeper (所有节点)

### 3.1 创建用户和安装目录

出于安全和权限管理考虑，建议创建专门的用户来运行ZooKeeper。

```bash
sudo groupadd zookeeper
sudo useradd -s /bin/nologin -g zookeeper -d /opt/zookeeper zookeeper
sudo mkdir -p /opt/zookeeper/data    # 数据目录
sudo mkdir -p /opt/zookeeper/logs    # 日志目录
sudo chown -R zookeeper:zookeeper /opt/zookeeper
```

### 3.2 解压ZooKeeper包

将下载的 `apache-zookeeper-3.8.3-bin.tar.gz` 移动到 `/opt/zookeeper` 目录下，并解压。

```bash
sudo mv apache-zookeeper-3.8.3-bin.tar.gz /opt/zookeeper/
cd /opt/zookeeper
sudo tar -zxvf apache-zookeeper-3.8.3-bin.tar.gz --strip-components=1
```
`--strip-components=1` 会去除压缩包内的顶层目录，直接将内容解压到 `/opt/zookeeper`。

### 3.3 分发ZooKeeper到其他节点

使用 `scp` 命令将 `/opt/zookeeper` 目录复制到其他从节点。

```bash
# 在master节点执行
scp -r /opt/zookeeper/ zookeeper@slave1:/opt/
scp -r /opt/zookeeper/ zookeeper@slave2:/opt/
```
**注意：** 如果你在安装ZooKeeper时没有使用专门的`zookeeper`用户，而是使用`root`用户，请确保在复制后修改文件的所有者和权限。

## 4. 配置ZooKeeper集群 (所有节点)

ZooKeeper的配置文件位于 `/opt/zookeeper/conf/zoo.cfg`。

### 4.1 创建 `zoo.cfg` 配置文件

```bash
sudo cp /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg
sudo vi /opt/zookeeper/conf/zoo.cfg
```

### 4.2 修改 `zoo.cfg`

主要修改以下配置项：

*   **`dataDir`：** 数据存储路径。
    ```
    dataDir=/opt/zookeeper/data
    ```
*   **`dataLogDir`：** 日志存储路径（可选，如果不设置则使用`dataDir`）。
    ```
    dataLogDir=/opt/zookeeper/logs
    ```
*   **`clientPort`：** 客户端连接端口，默认 `2181`。
    ```
    clientPort=2181
    ```
*   **`tickTime`：** ZooKeeper服务器之间或客户端与服务器之间维持心跳的时间间隔，单位毫秒。
    ```
    tickTime=2000
    ```
*   **`initLimit`：** Follower节点在启动时与Leader节点进行初始同步的超时时间，它由`tickTime`的数量表示。
    ```
    initLimit=10
    ```
*   **`syncLimit`：** Follower节点与Leader节点之间发送和接收消息的超时时间，它由`tickTime`的数量表示。
    ```
    syncLimit=5
    ```
*   **集群配置：** 在文件末尾添加集群中的所有ZooKeeper节点信息。格式为 `server.<myid>=<hostname>:<peer_port>:<leader_election_port>`。
    ```
    server.1=master-node:2888:3888
    server.2=slave-node-1:2888:3888
    server.3=slave-node-2:2888:3888
    ```
    *   `myid`：对应于当前节点的ID，必须与后面创建的`myid`文件内容一致。
    *   `hostname`：节点的IP地址或主机名。
    *   `peer_port`：Follower与Leader之间通信的端口。
    *   `leader_election_port`：用于Leader选举的端口。

### 4.3 创建 `myid` 文件 (每个节点不同)

在每个节点的 `dataDir` (即 `/opt/zookeeper/data`) 目录下创建一个 `myid` 文件，文件中只包含一个数字，这个数字就是该节点在 `zoo.cfg` 中配置的 `server.<myid>` 对应的 `myid`。

**在 `master-node` 上：**
```bash
sudo sh -c 'echo "1" > /opt/zookeeper/data/myid'
sudo chown zookeeper:zookeeper /opt/zookeeper/data/myid
```

**在 `slave-node-1` 上：**
```bash
sudo sh -c 'echo "2" > /opt/zookeeper/data/myid'
sudo chown zookeeper:zookeeper /opt/zookeeper/data/myid
```

**在 `slave-node-2` 上：**
```bash
sudo sh -c 'echo "3" > /opt/zookeeper/data/myid'
sudo chown zookeeper:zookeeper /opt/zookeeper/data/myid
```

## 5. 配置防火墙 (所有节点)

ZooKeeper默认使用 `2181` (客户端连接)、`2888` (Follower与Leader通信) 和 `3888` (Leader选举) 端口。所有节点都需要开放这些端口。

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-port=2181/tcp --permanent
sudo firewall-cmd --zone=public --add-port=2888/tcp --permanent
sudo firewall-cmd --zone=public --add-port=3888/tcp --permanent
sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 2181/tcp
sudo ufw allow 2888/tcp
sudo ufw allow 3888/tcp
sudo ufw enable           # 如果ufw未启用
sudo ufw status verbose
```

## 6. 启动ZooKeeper集群 (所有节点)

在所有节点上，使用ZooKeeper自带的脚本启动服务。

```bash
# 切换到zookeeper用户
sudo su - zookeeper

# 启动ZooKeeper
/opt/zookeeper/bin/zkServer.sh start

# 检查ZooKeeper状态
/opt/zookeeper/bin/zkServer.sh status
```
你可以看到 `Mode: follower` 或 `Mode: leader`，表示节点已成功加入集群。确保集群中有一个Leader节点，其他都是Follower节点。

### 6.1 配置Systemd服务 (可选，推荐)

为了方便管理和开机自启，可以将ZooKeeper配置为Systemd服务。

**创建Systemd服务文件（所有节点）：**
```bash
sudo vi /etc/systemd/system/zookeeper.service
```
将以下内容粘贴到文件中：

```ini
[Unit]
Description=Apache Zookeeper server
After=network.target

[Service]
Type=forking
User=zookeeper
Group=zookeeper
ExecStart=/opt/zookeeper/bin/zkServer.sh start
ExecStop=/opt/zookeeper/bin/zkServer.sh stop
ExecReload=/opt/zookeeper/bin/zkServer.sh restart
TimeoutStartSec=30
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**重载Systemd并启动ZooKeeper：**
```bash
sudo systemctl daemon-reload
sudo systemctl start zookeeper
sudo systemctl enable zookeeper
sudo systemctl status zookeeper
```

## 7. 验证ZooKeeper集群

连接到任一ZooKeeper客户端，验证集群是否正常工作。

```bash
# 连接到客户端
/opt/zookeeper/bin/zkCli.sh -server localhost:2181

# 在客户端输入命令
create /test "hello"
get /test
quit
```
如果你可以在一个节点上创建ZNode并在另一个节点上读取它，则表示集群正常工作。

## 8. 卸载ZooKeeper

如果需要卸载ZooKeeper：

```bash
sudo systemctl stop zookeeper         # 停止ZooKeeper服务
sudo systemctl disable zookeeper      # 取消开机自启
sudo rm /etc/systemd/system/zookeeper.service # 删除Systemd服务文件
sudo systemctl daemon-reload          # 重载Systemd配置
sudo rm -rf /opt/zookeeper            # 删除ZooKeeper安装目录
sudo userdel zookeeper                # 删除ZooKeeper用户
sudo groupdel zookeeper               # 删除ZooKeeper组
```

至此，你已成功在Linux集群环境中安装和部署了Apache ZooKeeper，并了解了如何进行基本的管理和验证。
