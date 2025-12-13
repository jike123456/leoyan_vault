# 第五章-08-RabbitMQ安装部署[单机软件]

RabbitMQ是使用Erlang语言开发的一个开源消息队列系统，它是AMQP（高级消息队列协议）的实现，常用于分布式系统中进行异步通信和任务解耦。本节将详细介绍在Linux系统上安装和部署RabbitMQ的步骤。

## 1. 环境准备

*   **操作系统：** CentOS 7/8 或 Ubuntu Server 18.04/20.04 (或更高版本)
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保服务器可以访问互联网以下载必要的软件包
*   **Erlang/OTP：** RabbitMQ依赖于Erlang运行时环境。

### 1.1 更新系统软件包

```bash
# 对于CentOS/RHEL
sudo yum update -y

# 对于Ubuntu/Debian
sudo apt update && sudo apt upgrade -y
```

### 1.2 安装Erlang/OTP

RabbitMQ要求特定版本的Erlang。建议通过官方存储库或专门的Erlang解决方案平台（如`erlang-solutions`）安装。

**对于CentOS/RHEL (使用`erlang-solutions`仓库安装较新版本Erlang)：**

1.  **导入Erlang Solutions的GPG密钥：**
    ```bash
    sudo rpm --import https://packages.erlang-solutions.com/rpm/erlang_solutions.asc
    ```
2.  **创建Erlang Solutions仓库文件：**
    ```bash
    sudo vi /etc/yum.repos.d/erlang-solutions.repo
    ```
    添加以下内容：
    ```
    [erlang-solutions]
    name=CentOS/$releasever - Erlang Solutions
    baseurl=https://packages.erlang-solutions.com/rpm/centos/$releasever
    gpgcheck=1
    gpgkey=https://packages.erlang-solutions.com/rpm/erlang_solutions.asc
    enabled=1
    ```
    注意：`$releasever` 会自动替换为你的CentOS版本号（如7或8）。
3.  **安装Erlang：**
    ```bash
    sudo yum install erlang -y
    ```

**对于Ubuntu/Debian (使用`erlang-solutions`仓库安装较新版本Erlang)：**

1.  **导入Erlang Solutions的GPG密钥：**
    ```bash
    wget https://packages.erlang-solutions.com/ubuntu/erlang_solutions.asc
    sudo apt-key add erlang_solutions.asc
    ```
2.  **添加Erlang Solutions仓库：**
    ```bash
    echo "deb https://packages.erlang-solutions.com/ubuntu $(lsb_release -cs) contrib" | sudo tee /etc/apt/sources.list.d/erlang-solutions.list
    ```
    `$(lsb_release -cs)` 会自动替换为你的Ubuntu版本代号（如`focal`）。
3.  **更新包列表并安装Erlang：**
    ```bash
    sudo apt update
    sudo apt install erlang -y
    ```

**验证Erlang版本：**
```bash
erl
# 进入Erlang shell后输入：
erlang:system_info(otp_release).
# 退出Erlang shell：
q().
```

## 2. 安装RabbitMQ

### 2.1 添加RabbitMQ存储库

由于系统自带的RabbitMQ版本可能较旧，建议使用官方的RPM/APT存储库。

**对于CentOS/RHEL：**

1.  **导入RabbitMQ GPG密钥：**
    ```bash
    sudo rpm --import https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey
    ```
2.  **添加RabbitMQ仓库文件：**
    ```bash
    sudo vi /etc/yum.repos.d/rabbitmq.repo
    ```
    添加以下内容：
    ```
    [rabbitmq_rabbitmq-server]
    name=rabbitmq_rabbitmq-server
    baseurl=https://packagecloud.io/rabbitmq/rabbitmq-server/el/$releasever/$basearch
    repo_gpgcheck=1
    gpgcheck=1
    enabled=1
    gpgkey=https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey
    sslverify=1
    sslcacert=/etc/pki/tls/certs/ca-bundle.crt
    ```
    如果 `/etc/pki/tls/certs/ca-bundle.crt` 不存在，可以尝试将其改为 `sslcacert=/etc/ssl/certs/ca-certificates.crt`。
3.  **安装RabbitMQ服务器：**
    ```bash
    sudo yum install rabbitmq-server -y
    ```

**对于Ubuntu/Debian：**

1.  **导入RabbitMQ GPG密钥：**
    ```bash
    curl -fsSL https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/rabbitmq.gpg
    ```
2.  **添加RabbitMQ仓库：**
    ```bash
    echo "deb [signed-by=/usr/share/keyrings/rabbitmq.gpg] https://packagecloud.io/rabbitmq/rabbitmq-server/ubuntu/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/rabbitmq.list
    ```
3.  **更新包列表并安装RabbitMQ服务器：**
    ```bash
    sudo apt update
    sudo apt install rabbitmq-server -y
    ```

## 3. 管理RabbitMQ服务

安装完成后，RabbitMQ服务通常会自动启动。

### 3.1 启动、停止、重启RabbitMQ

```bash
sudo systemctl start rabbitmq-server    # 启动RabbitMQ
sudo systemctl stop rabbitmq-server     # 停止RabbitMQ
sudo systemctl restart rabbitmq-server  # 重启RabbitMQ
sudo systemctl enable rabbitmq-server   # 设置开机自启
sudo systemctl status rabbitmq-server   # 查看RabbitMQ服务状态
```
确认服务状态为 `active (running)`。

## 4. 配置RabbitMQ Web管理界面 (Management Plugin)

RabbitMQ提供了一个Web管理界面，方便监控和管理。

### 4.1 启用管理插件

```bash
sudo rabbitmq-plugins enable rabbitmq_management
```
### 4.2 重启RabbitMQ服务

```bash
sudo systemctl restart rabbitmq-server
```

## 5. 配置防火墙

RabbitMQ的Web管理界面默认监听 `15672` 端口，AMQP协议默认监听 `5672` 端口。如果你的系统启用了防火墙，需要开放这些端口。

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-port=5672/tcp --permanent # AMQP端口
sudo firewall-cmd --zone=public --add-port=15672/tcp --permanent # 管理界面端口
sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 5672/tcp
sudo ufw allow 15672/tcp
sudo ufw enable           # 如果ufw未启用
sudo ufw status verbose
```

现在，你应该可以通过浏览器访问 `http://你的服务器IP或域名:15672` 来进入RabbitMQ管理界面。

## 6. 添加和管理用户

默认的RabbitMQ管理界面需要用户名和密码登录。默认情况下，有一个`guest`用户，但它只能从`localhost`访问。为了安全起见，我们应该创建一个新的管理员用户。

### 6.1 创建用户

```bash
sudo rabbitmqctl add_user <username> <password>
# 示例：
sudo rabbitmqctl add_user admin YourSecurePassword
```

### 6.2 设置用户标签 (tag)

为用户设置 `administrator` 标签，使其拥有管理权限。
```bash
sudo rabbitmqctl set_user_tags <username> administrator
# 示例：
sudo rabbitmqctl set_user_tags admin administrator
```

### 6.3 设置用户权限 (vhost)

为用户设置对默认虚拟主机 `/` 的所有权限。
```bash
sudo rabbitmqctl set_permissions -p / <username> ".*" ".*" ".*"
# 示例：
sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```
现在你可以使用新创建的 `admin` 用户和密码登录RabbitMQ管理界面。

## 7. 其他常用命令

*   **列出用户：**
    ```bash
    sudo rabbitmqctl list_users
    ```
*   **删除用户：**
    ```bash
    sudo rabbitmqctl delete_user <username>
    ```
*   **列出vhost：**
    ```bash
    sudo rabbitmqctl list_vhosts
    ```
*   **列出队列：**
    ```bash
    sudo rabbitmqctl list_queues
    ```
*   **集群状态：** (在集群环境中使用)
    ```bash
    sudo rabbitmqctl cluster_status
    ```

## 8. 卸载RabbitMQ

如果需要卸载RabbitMQ：

```bash
sudo systemctl stop rabbitmq-server         # 停止RabbitMQ服务
sudo systemctl disable rabbitmq-server      # 取消开机自启
# 对于CentOS/RHEL
sudo yum remove rabbitmq-server erlang -y
sudo rm -rf /var/lib/rabbitmq               # 删除数据目录
sudo rm /etc/yum.repos.d/rabbitmq.repo      # 删除仓库文件
sudo rm /etc/yum.repos.d/erlang-solutions.repo # 删除Erlang仓库文件

# 对于Ubuntu/Debian
sudo apt autoremove --purge rabbitmq-server erlang -y
sudo rm -rf /var/lib/rabbitmq               # 删除数据目录
sudo rm /etc/apt/sources.list.d/rabbitmq.list # 删除仓库文件
sudo rm /etc/apt/sources.list.d/erlang-solutions.list # 删除Erlang仓库文件
sudo rm /usr/share/keyrings/rabbitmq.gpg # 删除GPG密钥
sudo rm erlang_solutions.asc # 删除GPG密钥
```

至此，你已成功在Linux系统上安装并部署了RabbitMQ，并了解了如何进行基本的管理和用户配置。
