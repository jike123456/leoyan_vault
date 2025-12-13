# 第五章-16-HBase集群部署[集群化软件]

Apache HBase是一个开源的、分布式的、可扩展的、列式存储的NoSQL数据库，构建在HDFS之上，并提供对Hadoop数据的随机、实时读/写访问。它非常适合存储半结构化和非结构化的大型数据集。本节将详细介绍如何在Linux集群环境中部署HBase集群。

## 1. 环境准备

*   **操作系统：** 至少三台Linux服务器（CentOS 7/8 或 Ubuntu Server 18.04/20.04），其中一台作为HMaster，其他作为RegionServer。
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保所有节点之间可以互相访问，并根据**第五章-11-集群化软件安装前置准备**配置好主机名和SSH免密码登录。
*   **Java环境：** HBase依赖于Java运行环境（JDK）。
*   **Hadoop集群：** HBase需要一个运行正常的Hadoop HDFS集群作为其底层存储。参考**第五章-15-Hadoop集群部署[集群化软件]**。
*   **ZooKeeper集群：** HBase依赖于ZooKeeper进行分布式协调。HBase通常会启动自带的ZooKeeper，但生产环境建议使用独立的ZooKeeper集群。参考**第五章-13-Zookeeper安装部署[集群化软件]**。

### 1.1 安装Java开发工具包 (JDK)

所有HBase节点都需要安装JDK。HBase通常与Hadoop版本兼容的JDK。

**配置JAVA_HOME环境变量（在所有节点上）：**
```bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64 # 根据实际路径修改
export PATH=$PATH:$JAVA_HOME/bin
```
然后 `source /etc/profile` 使其生效。

## 2. 下载HBase

访问Apache HBase官网（hbase.apache.org/downloads.html）获取最新稳定版的二进制包。

```bash
# 在一个节点上下载，然后通过scp分发到其他节点
wget https://dlcdn.apache.org/hbase/2.5.8/hbase-2.5.8-bin.tar.gz # 请替换为最新版本
```

## 3. 安装HBase (所有节点)

### 3.1 创建用户和安装目录

出于安全和权限管理考虑，建议创建专门的用户来运行HBase。

```bash
sudo groupadd hbase
sudo useradd -s /bin/bash -g hbase -d /opt/hbase hbase
sudo mkdir -p /opt/hbase
sudo chown -R hbase:hbase /opt/hbase
```

### 3.2 解压HBase包

将下载的 `hbase-2.5.8-bin.tar.gz` 移动到 `/opt/hbase` 目录下，并解压。

```bash
sudo mv hbase-2.5.8-bin.tar.gz /opt/hbase/
cd /opt/hbase
sudo tar -zxvf hbase-2.5.8-bin.tar.gz --strip-components=1
```
`--strip-components=1` 会去除压缩包内的顶层目录，直接将内容解压到 `/opt/hbase`。

### 3.3 分发HBase到其他节点

使用 `scp` 命令将 `/opt/hbase` 目录复制到所有RegionServer节点。

```bash
# 在master节点执行
scp -r /opt/hbase/ hbase@slave1:/opt/
scp -r /opt/hbase/ hbase@slave2:/opt/
```

## 4. 配置HBase集群 (所有节点)

HBase的配置文件主要位于 `/opt/hbase/conf/` 目录下。

### 4.1 修改 `hbase-env.sh` (所有节点)

编辑 `/opt/hbase/conf/hbase-env.sh`：
```bash
sudo vi /opt/hbase/conf/hbase-env.sh
```
主要修改以下配置项：

*   **`JAVA_HOME`：**
    ```bash
    export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64 # 根据实际路径修改
    ```
*   **`HBASE_MANAGES_ZK`：** 如果使用独立的ZooKeeper集群，设置为 `false`。
    ```bash
    export HBASE_MANAGES_ZK=false
    ```

### 4.2 修改 `hbase-site.xml` (所有节点)

编辑 `/opt/hbase/conf/hbase-site.xml`：
```bash
sudo vi /opt/hbase/conf/hbase-site.xml
```
```xml
<configuration>
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value> <!-- 开启分布式模式 -->
    </property>
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://master-node:9000/hbase</value> <!-- HBase数据在HDFS上的根目录 -->
    </property>
    <property>
        <name>hbase.zookeeper.quorum</name>
        <value>master-node:2181,slave1:2181,slave2:2181</value> <!-- ZooKeeper集群地址 -->
    </property>
    <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/opt/zookeeper/data</value> <!-- 如果HBASE_MANAGES_ZK为true，ZooKeeper数据目录 -->
    </property>
    <property>
        <name>hbase.tmp.dir</name>
        <value>/opt/hbase/tmp</value> <!-- HBase临时目录 -->
    </property>
    <property>
        <name>hbase.master.port</name>
        <value>16000</value> <!-- HMaster RPC端口 -->
    </property>
    <property>
        <name>hbase.master.info.port</name>
        <value>16010</value> <!-- HMaster Web UI端口 -->
    </property>
</configuration>
```

### 4.3 配置 `regionservers` 文件 (仅HMaster节点)

编辑 `/opt/hbase/conf/regionservers`，列出所有RegionServer节点的主机名。
```bash
sudo vi /opt/hbase/conf/regionservers
```
```
slave1
slave2
```

## 5. 配置HBase环境变量 (所有节点)

在 `/etc/profile` 或 `~/.bashrc` 中添加HBase相关的环境变量：
```bash
export HBASE_HOME=/opt/hbase
export PATH=$PATH:$HBASE_HOME/bin
```
然后 `source /etc/profile` 使其生效。

## 6. 启动HBase集群

**重要：确保Hadoop HDFS和ZooKeeper集群已正常运行。**

### 6.1 启动HBase (仅HMaster节点)

```bash
# 切换到hbase用户
sudo su - hbase
start-hbase.sh
```
这会启动HMaster和所有RegionServer。

### 6.2 验证进程

在HMaster节点上执行 `jps` 命令，应该看到 HMaster 进程。
在RegionServer节点上执行 `jps` 命令，应该看到 HRegionServer 进程。

### 6.3 访问Web UI

*   **HBase Master UI：** `http://master-node:16010`

## 7. 配置防火墙 (所有节点)

需要开放HBase集群使用的端口。

**主要端口：**
*   `16000`: HMaster RPC端口
*   `16010`: HMaster Web UI端口
*   `16020`: HRegionServer RPC端口
*   `16030`: HRegionServer Web UI端口

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-port=16000/tcp --permanent
sudo firewall-cmd --zone=public --add-port=16010/tcp --permanent
sudo firewall-cmd --zone=public --add-port=16020/tcp --permanent
sudo firewall-cmd --zone=public --add-port=16030/tcp --permanent
sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 16000/tcp
sudo ufw allow 16010/tcp
sudo ufw allow 16020/tcp
sudo ufw allow 16030/tcp
sudo ufw enable           # 如果ufw未启用
sudo ufw status verbose
```

## 8. 运行HBase Shell示例

连接到HBase Shell，进行一些基本操作。

```bash
# 切换到hbase用户
sudo su - hbase
hbase shell
```
在HBase Shell中：
```
create 'test_table', 'cf'               # 创建表
list                                    # 列出表
put 'test_table', 'row1', 'cf:name', 'value1' # 插入数据
get 'test_table', 'row1'                # 获取数据
scan 'test_table'                       # 扫描表
disable 'test_table'
drop 'test_table'
exit
```

## 9. 停止HBase集群 (仅HMaster节点)

```bash
# 切换到hbase用户
sudo su - hbase
stop-hbase.sh
```

## 10. 卸载HBase

如果需要卸载HBase：

```bash
sudo systemctl stop hbase* # 如果配置了Systemd服务
sudo rm -rf /opt/hbase            # 删除HBase安装目录
sudo userdel hbase                # 删除HBase用户
sudo groupdel hbase               # 删除HBase组
# 清理环境变量 (从 /etc/profile 或 ~/.bashrc 中删除)
```

至此，你已成功在Linux集群环境中安装和部署了Apache HBase，并了解了如何进行基本的管理和测试。
