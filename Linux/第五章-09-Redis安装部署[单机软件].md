# 第五章-09-Redis安装部署[单机软件]

Redis是一个开源（BSD许可）的内存数据结构存储，可用作数据库、缓存和消息代理。它支持多种数据结构，如字符串、哈希、列表、集合、有序集合等，以其高性能、丰富的功能和简洁的API受到广泛欢迎。本节将详细介绍在Linux系统上安装和部署Redis的步骤。

## 1. 环境准备

*   **操作系统：** CentOS 7/8 或 Ubuntu Server 18.04/20.04 (或更高版本)
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保服务器可以访问互联网以下载必要的软件包

### 1.1 更新系统软件包并安装编译依赖

Redis通常通过源码编译安装，需要安装一些编译工具和依赖库。

**对于CentOS/RHEL：**
```bash
sudo yum update -y
sudo yum install gcc make -y
```

**对于Ubuntu/Debian：**
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install build-essential tcl -y # tcl用于运行Redis自带的测试脚本
```

## 2. 下载Redis

访问Redis官网（redis.io/download）获取最新稳定版的Redis源码包。

```bash
# 这里以Redis 7.0.x为例，请根据官网最新稳定版修改URL
wget https://download.redis.io/releases/redis-7.0.12.tar.gz
```

## 3. 安装Redis

### 3.1 解压源码包

```bash
tar -zxvf redis-7.0.12.tar.gz
cd redis-7.0.12
```

### 3.2 编译和安装

```bash
make
sudo make install
```
`make install` 命令会将编译好的可执行文件（`redis-server`, `redis-cli`, `redis-benchmark`, `redis-check-rdb`, `redis-check-aof`）拷贝到 `/usr/local/bin` 目录下，这样就可以在任何地方直接运行这些命令。

### 3.3 运行测试 (可选但推荐)

```bash
make test
```
这会运行Redis自带的单元测试，确保编译正常。

## 4. 配置Redis

为了更好地管理Redis服务，我们需要将其配置为Systemd服务，并对配置文件进行一些调整。

### 4.1 创建Redis配置文件目录和数据目录

```bash
sudo mkdir /etc/redis
sudo mkdir /var/lib/redis
```

### 4.2 复制配置文件

将源码包中的`redis.conf`配置文件复制到`/etc/redis/`目录下。

```bash
sudo cp redis.conf /etc/redis/
```

### 4.3 修改Redis配置文件

编辑 `/etc/redis/redis.conf` 文件。

```bash
sudo vi /etc/redis/redis.conf
```
主要修改以下几项：
*   **`daemonize yes`：** 将 `daemonize no` 改为 `daemonize yes`，让Redis以后台守护进程方式运行。
*   **`bind 127.0.0.1`：** 默认只允许本地访问。如果需要远程访问，可以修改为 `bind 0.0.0.0` 或注释掉该行。**注意：** 生产环境中，更推荐通过防火墙限制访问或使用SSH隧道。
*   **`protected-mode yes`：** 保持 `protected-mode yes`，这是为了安全，如果 `bind` 不为 `127.0.0.1` 且没有设置密码，则只允许本地连接。
*   **`port 6379`：** 默认端口6379，如果需要修改，请在这里设置。
*   **`logfile "/var/log/redis/redis.log"`：** 建议设置日志文件路径。
*   **`dir /var/lib/redis`：** 将 `dir ./` 改为 `dir /var/lib/redis`，指定Redis数据存放目录。
*   **`requirepass foobared`：** 设置密码，将 `# requirepass foobared` 前的 `#` 去掉，并将 `foobared` 改为你的强密码。
    ```
    requirepass YourRedisSecurePassword
    ```
*   **`maxmemory <bytes>`：** 建议设置最大内存限制，避免Redis占用过多内存导致系统不稳定。
*   **`maxmemory-policy noeviction`：** 当达到最大内存限制时的淘汰策略。

### 4.4 创建日志目录和文件（如果设置了日志路径）

```bash
sudo mkdir /var/log/redis
sudo touch /var/log/redis/redis.log
sudo chown redis:redis /var/log/redis/redis.log # 如果Redis运行用户不是root
```

### 4.5 创建Redis运行用户（推荐）

出于安全考虑，不建议使用root用户运行Redis。

```bash
sudo adduser --system --group --no-create-home redis
sudo chown redis:redis /var/lib/redis
```

### 4.6 创建Systemd服务文件

为了方便管理Redis，我们将其配置为一个Systemd服务。

```bash
sudo vi /etc/systemd/system/redis.service
```
将以下内容粘贴到文件中：

```ini
[Unit]
Description=Redis persistent key-value database
After=network.target

[Service]
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
User=redis  # 使用redis用户运行
Group=redis # 使用redis组运行
Type=forking
TimeoutStopSec=10
Restart=always
RestartSec=1
LimitNOFILE=65536
LimitNPROC=65536

[Install]
WantedBy=multi-user.target
```
**注意：** `User=redis` 和 `Group=redis` 确保Redis以我们创建的用户和组运行。

### 4.7 重载Systemd并启动Redis

```bash
sudo systemctl daemon-reload # 重载Systemd配置
sudo systemctl start redis   # 启动Redis服务
sudo systemctl enable redis  # 设置Redis开机自启
sudo systemctl status redis  # 检查Redis服务状态
```
确认服务状态为 `active (running)`。

## 5. 配置防火墙

Redis默认监听 `6379` 端口。如果你的系统启用了防火墙，需要开放该端口。

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-port=6379/tcp --permanent
sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 6379/tcp
sudo ufw enable           # 如果ufw未启用
sudo ufw status verbose
```

## 6. 使用Redis客户端

通过 `redis-cli` 命令连接到Redis服务器。

```bash
redis-cli -a YourRedisSecurePassword # 如果设置了密码
# 或者先登录，再认证
# redis-cli
# auth YourRedisSecurePassword
```
连接成功后，可以尝试一些Redis命令：
```
ping # 应该返回 PONG
set mykey "Hello Redis"
get mykey # 应该返回 "Hello Redis"
```

## 7. 卸载Redis

如果需要卸载Redis：

```bash
sudo systemctl stop redis         # 停止Redis服务
sudo systemctl disable redis      # 取消开机自启
sudo rm /etc/systemd/system/redis.service # 删除Systemd服务文件
sudo systemctl daemon-reload      # 重载Systemd配置
sudo rm -rf /etc/redis            # 删除配置文件目录
sudo rm -rf /var/lib/redis        # 删除数据目录
sudo rm -rf /var/log/redis        # 删除日志目录
sudo rm -rf /usr/local/bin/redis* # 删除Redis可执行文件
sudo userdel redis                # 删除Redis用户
sudo groupdel redis               # 删除Redis组
sudo rm -rf /opt/redis-7.0.12     # 删除源码目录 (如果还在)
```

至此，你已成功在Linux系统上安装并部署了Redis，并了解了如何进行基本的管理和使用。
