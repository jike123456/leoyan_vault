# 第五章-02-MySQL5.7在CentOS安装[单机软件]

MySQL是世界上最流行的开源关系型数据库管理系统。本节将详细介绍在CentOS系统上安装MySQL 5.7社区版的步骤。

## 1. 环境准备

*   **操作系统：** CentOS 7 或 CentOS 8
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保服务器可以访问互联网以下载必要的软件包

### 1.1 更新系统软件包

在安装任何新软件之前，始终建议更新系统到最新状态。
```bash
sudo yum update -y
```

### 1.2 卸载可能存在的旧版本MySQL或其他数据库

如果你的系统上已经安装了MariaDB或其他MySQL版本，建议先将其卸载，以避免冲突。
```bash
sudo yum remove mariadb-libs -y # 卸载MariaDB相关包
sudo rpm -qa | grep mysql # 检查是否还有其他MySQL安装包
sudo yum remove mysql* -y # 如果有，卸载所有MySQL相关包
```

## 2. 下载并安装MySQL Yum存储库

MySQL官方提供了一个Yum存储库，可以方便地通过`yum`命令安装MySQL。

### 2.1 下载存储库RPM包

访问MySQL官网（dev.mysql.com/downloads/repo/yum/）获取最新版本的RPM包下载链接。这里以MySQL 5.7为例。
```bash
wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
```
**注意：**
*   `el7` 适用于 CentOS 7/RHEL 7。
*   `el8` 适用于 CentOS 8/RHEL 8。请根据你的系统版本选择正确的RPM包。

### 2.2 安装存储库

```bash
sudo rpm -Uvh mysql57-community-release-el7-11.noarch.rpm
```
安装成功后，`/etc/yum.repos.d/` 目录下会生成 `mysql-community.repo` 等文件。

### 2.3 检查MySQL版本配置（可选）

默认情况下，MySQL Yum存储库会启用最新稳定版。如果你想安装特定版本（例如确保是5.7），可以检查并修改repo文件。

编辑 `/etc/yum.repos.d/mysql-community.repo` 文件：
```bash
sudo vi /etc/yum.repos.d/mysql-community.repo
```
找到 `[mysql57-community]` 部分，确保 `enabled=1`。
找到 `[mysql80-community]` 或其他版本部分，确保 `enabled=0`。

## 3. 安装MySQL服务器

一切准备就绪后，现在可以通过`yum`命令安装MySQL服务器。

```bash
sudo yum install mysql-community-server -y
```
这个命令会安装MySQL服务器及其所有依赖项。

## 4. 启动MySQL服务并设置开机自启

安装完成后，启动MySQL服务并将其设置为开机自启。

```bash
sudo systemctl start mysqld        # 启动MySQL服务
sudo systemctl enable mysqld       # 设置MySQL开机自启
sudo systemctl status mysqld       # 检查MySQL服务状态
```
确认服务状态为 `active (running)`。

## 5. 初始化MySQL（重要步骤！）

MySQL 5.7在首次启动时会进行初始化，并为`root`用户生成一个临时密码。这个密码非常重要，你需要找到它来登录MySQL并设置新密码。

### 5.1 获取临时密码

临时密码通常存储在MySQL的日志文件中：
```bash
sudo grep 'temporary password' /var/log/mysqld.log
```
你将看到类似 `root@localhost: <临时密码>` 的输出。记下这个临时密码。
[grep](第二章-11-grep-wc-管道符.md#^ebd24d)
### 5.2 登录MySQL并修改`root`密码

==使用临时密码登录MySQL==：
```bash
mysql -uroot -p
```
输入你刚刚找到的临时密码。

登录成功后，MySQL会强制你==修改`root`密码。==MySQL 5.7默认启用了密码策略验证插件（`validate_password`），要求密码必须包含大小写字母、数字、特殊字符，并且长度至少为8位。

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourNewSecurePassword123!';
```
**注意：** 请将 `'YourNewSecurePassword123!'` 替换为你自己的强密码。

如果遇到密码策略问题，可以暂时降低密码策略等级，修改完密码后再调回去（不推荐在生产环境这样做）。
```sql
# 查看密码策略
SHOW VARIABLES LIKE 'validate_password%';

# 临时修改密码策略 (仅供测试或学习环境使用)
set global validate_password_policy=0;   # 设为LOW，只检查长度
set global validate_password_length=6;  # 设为最小长度，例如6

# 再次修改密码（使用简单密码）
ALTER USER 'root'@'localhost' IDENTIFIED BY 'newpassword';

# 恢复密码策略 (强烈建议在生产环境恢复)
set global validate_password_policy=1; # 设为MEDIUM
set global validate_password_length=8; # 设为8
```

## 6. 配置远程访问（可选）

默认情况下，MySQL `root` 用户只能从 `localhost` 访问。如果需要从其他机器连接MySQL，需要创建新的用户或修改`root`用户的权限。

### 6.1 创建新用户并授权

这是一个更安全的方式，避免直接使用`root`进行远程连接。
```sql
# 创建一个名为 'remote_user'，密码为 'YourRemotePassword!' 的用户，并允许从任何IP连接
CREATE USER 'remote_user'@'%' IDENTIFIED BY 'YourRemotePassword!';

# 授予该用户所有数据库的所有权限（根据实际需求调整权限）
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%';

# 刷新权限
FLUSH PRIVILEGES;
```
**注意：** `'%'` 表示允许从任何主机连接，你也可以指定特定的IP地址，例如 `'192.168.1.100'`。

### 6.2 开放防火墙端口

如果你的CentOS系统启用了`firewalld`（默认启用），需要开放MySQL的默认端口`3306`。
```bash
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
```

## 7. 其他常用配置

### 7.1 修改MySQL配置文件

MySQL的主配置文件通常是 `/etc/my.cnf`。你可以在这里调整MySQL的各种参数，例如字符集、缓冲区大小等。

编辑配置文件：
```bash
sudo vi /etc/my.cnf
```
例如，配置默认字符集为 `utf8mb4`：
```ini
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
```
修改后，需要重启MySQL服务使配置生效：
```bash
sudo systemctl restart mysqld
```

### 7.2 卸载MySQL

如果需要卸载MySQL：
```bash
sudo systemctl stop mysqld                  # 停止MySQL服务
sudo yum remove mysql-community-server mysql-community-client mysql-community-common mysql-community-libs -y # 卸载软件包
sudo rm -rf /var/lib/mysql                  # 删除数据目录
sudo rm -rf /etc/my.cnf                     # 删除配置文件
sudo rm -rf /var/log/mysqld.log             # 删除日志文件
sudo rm -rf /etc/yum.repos.d/mysql-community* # 删除repo文件
```

至此，你已成功在CentOS上安装并配置了MySQL 5.7数据库。你可以通过命令行客户端或图形化工具连接到数据库，并开始你的数据库操作。
