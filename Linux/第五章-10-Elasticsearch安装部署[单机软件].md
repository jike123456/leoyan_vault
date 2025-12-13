# 第五章-10-Elasticsearch安装部署[单机软件]

Elasticsearch是一个开源的、分布式的、RESTful风格的搜索和分析引擎，能够解决近实时地存储、搜索和分析大量数据的问题。它通常与Kibana（数据可视化工具）、Logstash（数据收集引擎）一起组成ELK技术栈，广泛应用于日志分析、全文搜索、指标分析等场景。本节将详细介绍在Linux系统上安装和部署单机版Elasticsearch的步骤。

## 1. 环境准备

*   **操作系统：** CentOS 7/8 或 Ubuntu Server 18.04/20.04 (或更高版本)
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保服务器可以访问互联网以下载必要的软件包
*   **Java环境：** Elasticsearch依赖于Java运行环境（JDK）。不同版本的Elasticsearch对JDK版本有要求，请查阅官方文档。通常建议安装OpenJDK。

### 1.1 安装Java开发工具包 (JDK)

Elasticsearch 8.x需要Java 17或更高版本。这里以OpenJDK 17为例。

**对于CentOS/RHEL：**
```bash
sudo yum install java-17-openjdk-devel -y
# 验证Java版本
java -version
```

**对于Ubuntu/Debian：**
```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
# 验证Java版本
java -version
```

## 2. 下载并安装Elasticsearch

建议通过官方APT/YUM存储库安装Elasticsearch，这能简化管理和更新。

### 2.1 添加Elasticsearch存储库

**对于CentOS/RHEL：**

1.  **导入Elasticsearch GPG密钥：**
    ```bash
    sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
    ```
2.  **创建Elasticsearch仓库文件：**
    ```bash
    sudo vi /etc/yum.repos.d/elasticsearch.repo
    ```
    添加以下内容（请根据Elasticsearch版本调整）：
    ```
    [elasticsearch-8.x]
    name=Elasticsearch repository for 8.x packages
    baseurl=https://artifacts.elastic.co/packages/8.x/yum
    gpgcheck=1
    gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
    enabled=1
    autorefresh=1
    type=rpm-md
    ```
    请注意 `8.x` 部分，如果安装其他版本请相应修改。

3.  **安装Elasticsearch：**
    ```bash
    sudo yum install elasticsearch -y
    ```

**对于Ubuntu/Debian：**

1.  **安装`apt-transport-https`：**
    ```bash
    sudo apt install apt-transport-https -y
    ```
2.  **导入Elasticsearch GPG密钥：**
    ```bash
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
    ```
3.  **添加Elasticsearch仓库文件：**
    ```bash
    echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
    ```
    请注意 `8.x` 部分，如果安装其他版本请相应修改。

4.  **更新APT包列表并安装Elasticsearch：**
    ```bash
    sudo apt update
    sudo apt install elasticsearch -y
    ```

## 3. 配置Elasticsearch

安装完成后，Elasticsearch的配置文件位于 `/etc/elasticsearch/elasticsearch.yml`。

### 3.1 修改主配置文件

```bash
sudo vi /etc/elasticsearch/elasticsearch.yml
```
主要修改以下配置项：

*   **`cluster.name`：** 集群名称，保持默认或自定义一个有意义的名称。单机部署也可以设置。
    ```yaml
    cluster.name: my-application
    ```
*   **`node.name`：** 节点名称，默认是主机名。
    ```yaml
    node.name: node-1
    ```
*   **`path.data`：** 数据存储路径。默认是 `/var/lib/elasticsearch`。
*   **`path.logs`：** 日志存储路径。默认是 `/var/log/elasticsearch`。
*   **`network.host`：** 绑定IP地址。默认只绑定`localhost`。如果需要其他机器访问，可以设置为 `0.0.0.0` 或指定一个IP。
    ```yaml
    network.host: 0.0.0.0
    ```
*   **`http.port`：** HTTP端口，默认`9200`。
*   **`transport.port`：** 节点间通信端口，默认`9300`。
*   **`discovery.seed_hosts`：** 集群发现的主机列表，单机部署时无需配置或设置为 `["127.0.0.1"]`。
*   **`xpack.security.enabled: true`** (Elasticsearch 8.x 默认启用安全功能，包括认证和TLS/SSL加密)
    初次启动时，Elasticsearch会自动生成 `elastic` 用户的密码和用于HTTPS通信的TLS证书。

### 3.2 调整系统参数

Elasticsearch需要较高的文件句柄数和虚拟内存映射数。

*   **修改文件句柄数：**
    ```bash
    sudo vi /etc/security/limits.conf
    ```
    在文件末尾添加（用户名为Elasticsearch运行用户，通常是`elasticsearch`）：
    ```
    elasticsearch - nofile 65536
    elasticsearch - memlock unlimited
    ```
    `memlock unlimited` 用于防止交换内存，但可能需要禁用Swap分区。

*   **修改虚拟内存映射数：**
    ```bash
    sudo vi /etc/sysctl.conf
    ```
    在文件末尾添加：
    ```
    vm.max_map_count=262144
    ```
    然后执行 `sudo sysctl -p` 使其生效。

### 3.3 禁用Swap (推荐)

为防止Elasticsearch使用Swap内存影响性能，建议禁用Swap。

```bash
sudo swapoff -a
sudo vi /etc/fstab # 注释掉或删除Swap分区行
```

## 4. 启动Elasticsearch服务并设置开机自启

安装完成后，启动Elasticsearch服务并将其设置为开机自启。

```bash
sudo systemctl start elasticsearch        # 启动Elasticsearch服务
sudo systemctl enable elasticsearch       # 设置Elasticsearch开机自启
sudo systemctl status elasticsearch       # 检查Elasticsearch服务状态
```
确认服务状态为 `active (running)`。

**注意：首次启动时**
Elasticsearch 8.x在首次启动时会自动生成一些安全配置信息，包括`elastic`用户的密码和enrollment token。这些信息会输出到启动日志中（通常是`/var/log/elasticsearch/elasticsearch.log`）。

你需要查找并保存以下信息：
*   **`elastic` 用户的密码：** `Password for the elastic user...`
*   **`enrollment token`：** `To continue with the setup process, use the following command...` (用于连接Kibana或添加新节点)

## 5. 配置防火墙

Elasticsearch默认监听 `9200` (HTTP) 和 `9300` (节点间通信) 端口。如果你的系统启用了防火墙，需要开放这些端口。

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-port=9200/tcp --permanent
sudo firewall-cmd --zone=public --add-port=9300/tcp --permanent
sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 9200/tcp
sudo ufw allow 9300/tcp
sudo ufw enable           # 如果ufw未启用
sudo ufw status verbose
```

## 6. 验证Elasticsearch

现在，你应该可以通过 `curl` 命令验证Elasticsearch是否正常运行。
**注意：** Elasticsearch 8.x默认启用HTTPS，你需要使用 `curl -u elastic:<password> --cacert <path_to_ca_cert> https://<your_ip>:9200` 来访问。

```bash
# 对于localhost访问 (如果未修改network.host)
curl -X GET "localhost:9200"

# 如果配置了HTTPS和认证 (Elasticsearch 8.x 默认)
# 首先找到CA证书路径，通常在 /etc/elasticsearch/certs/http_ca.crt
# 然后使用之前获取的elastic密码
curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic:<your_elastic_password> https://<your_ip_or_localhost>:9200
```
如果返回Elasticsearch的版本和集群信息，则表示安装成功。

## 7. 卸载Elasticsearch

如果需要卸载Elasticsearch：

```bash
sudo systemctl stop elasticsearch         # 停止Elasticsearch服务
sudo systemctl disable elasticsearch      # 取消开机自启

# 对于CentOS/RHEL
sudo yum remove elasticsearch -y
sudo rm -rf /etc/elasticsearch            # 删除配置文件
sudo rm -rf /var/lib/elasticsearch        # 删除数据目录
sudo rm -rf /var/log/elasticsearch        # 删除日志目录
sudo rm /etc/yum.repos.d/elasticsearch.repo # 删除仓库文件

# 对于Ubuntu/Debian
sudo apt autoremove --purge elasticsearch -y
sudo rm -rf /etc/elasticsearch            # 删除配置文件
sudo rm -rf /var/lib/elasticsearch        # 删除数据目录
sudo rm -rf /var/log/elasticsearch        # 删除日志目录
sudo rm /etc/apt/sources.list.d/elastic-8.x.list # 删除仓库文件
sudo apt-key del <GPG_KEY_ID>             # 删除GPG密钥 (使用 apt-key list 查找ID)
```

至此，你已成功在Linux系统上安装并部署了单机版Elasticsearch。接下来你可以考虑安装Kibana进行数据可视化和管理。
