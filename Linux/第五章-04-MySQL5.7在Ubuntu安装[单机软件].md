# 第五章-04-MySQL5.7在Ubuntu安装[单机软件]

MySQL是世界上最流行的开源关系型数据库管理系统。本节将详细介绍在Ubuntu系统上安装MySQL 5.7社区版的步骤。

## 1. 环境准备

*   **操作系统：** Ubuntu Server 16.04/18.04/20.04 (或更高版本)
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保服务器可以访问互联网以下载必要的软件包

### 1.1 更新系统软件包

在安装任何新软件之前，始终建议更新系统到最新状态。
```bash
sudo apt update -y
sudo apt upgrade -y
```

### 1.2 卸载可能存在的旧版本MySQL或其他数据库

如果你的系统上已经安装了MariaDB或其他MySQL版本，建议先将其卸载，以避免冲突。
```bash
sudo apt autoremove --purge mysql-server mysql-client mysql-common -y # 卸载MySQL相关包
sudo apt autoremove --purge mariadb-server mariadb-client mariadb-common -y # 卸载MariaDB相关包
sudo rm -rf /etc/mysql /var/lib/mysql # 删除残留文件和数据目录
```

## 2. 下载并安装MySQL APT存储库

MySQL官方提供了一个APT存储库，可以方便地通过`apt`命令安装MySQL。

### 2.1 下载APT存储库配置包

访问MySQL官网（dev.mysql.com/downloads/repo/apt/）获取最新版本的`.deb`包下载链接。这里以MySQL 5.7为例。
```bash
wget https://dev.mysql.com/get/mysql-apt-config_0.8.16-1_all.deb
```
**注意：** 请根据你的Ubuntu版本和所需的MySQL版本选择正确的`.deb`包。

### 2.2 安装存储库配置包

```bash
sudo dpkg -i mysql-apt-config_0.8.16-1_all.deb
```
执行此命令后，会弹出一个交互式配置界面。在其中，你需要选择要安装的MySQL版本。
1.  **Select the MySQL product you wish to configure:** 使用方向键选择 `MySQL Server & Cluster`。
2.  在下一页选择 `mysql-5.7`。
3.  选择 `Ok` 完成配置。

### 2.3 更新APT包列表

配置APT存储库后，需要更新包列表以包含MySQL的软件包。
```bash
sudo apt update
```

## 3. 安装MySQL服务器

一切准备就绪后，现在可以通过`apt`命令安装MySQL服务器。

```bash
sudo apt install mysql-server -y
```
在安装过程中，系统可能会提示你设置MySQL `root` 用户的密码。请设置一个强密码并牢记。

## 4. 启动MySQL服务并设置开机自启

安装完成后，MySQL服务通常会自动启动。你也可以手动启动和检查状态。

```bash
sudo systemctl start mysql        # 启动MySQL服务
sudo systemctl enable mysql       # 设置MySQL开机自启
sudo systemctl status mysql       # 检查MySQL服务状态
```
确认服务状态为 `active (running)`。

## 5. 首次安全设置

安装完成后，建议运行MySQL提供的安全脚本来增强数据库的安全性。
```bash
sudo mysql_secure_installation
```
此脚本会引导你完成以下安全设置：
1.  **VALIDATE PASSWORD COMPONENT：** 提示你是否启用密码强度验证插件。建议启用。
2.  **Change the root password?** 如果你在安装时已经设置了密码，可以选择 `n`。如果没有，则需要设置。
3.  **Remove anonymous users?** 建议 `Y`。
4.  **Disallow root login remotely?** 建议 `Y`，以防止`root`用户远程登录，增加安全性。
5.  **Remove test database and access to it?** 建议 `Y`。
6.  **Reload privilege tables now?** 建议 `Y`。

## 6. 远程访问配置（可选）

默认情况下，MySQL `root` 用户可能只能从 `localhost` 访问。如果需要从其他机器连接MySQL，需要创建新的用户或修改配置。

### 6.1 修改MySQL绑定地址

编辑MySQL配置文件 `/etc/mysql/mysql.conf.d/mysqld.cnf`。
```bash
sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
```
找到 `bind-address` 行，将其从 `127.0.0.1` 修改为 `0.0.0.0`，或注释掉这一行，允许所有IP地址连接。
```ini
# bind-address = 127.0.0.1
bind-address = 0.0.0.0
```
修改后，重启MySQL服务：
```bash
sudo systemctl restart mysql
```

### 6.2 创建远程用户并授权

这是一个更安全的方式，避免直接使用`root`进行远程连接。
登录MySQL：
```bash
mysql -uroot -p
# 输入root密码
```
在MySQL命令行中执行：
```sql
# 创建一个名为 'remote_user'，密码为 'YourRemotePassword!' 的用户，并允许从任何IP连接
CREATE USER 'remote_user'@'%' IDENTIFIED BY 'YourRemotePassword!';

# 授予该用户所有数据库的所有权限（根据实际需求调整权限）
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%';

# 刷新权限
FLUSH PRIVILEGES;
```
**注意：** `'%'` 表示允许从任何主机连接，你也可以指定特定的IP地址，例如 `'192.168.1.100'`。

### 6.3 开放防火墙端口

如果你的Ubuntu系统启用了`ufw`防火墙（默认启用），需要开放MySQL的默认端口`3306`。
```bash
sudo ufw allow 3306/tcp
sudo ufw enable       # 确保ufw已启用
sudo ufw status verbose # 检查防火墙状态
```

## 7. 其他常用配置

### 7.1 修改MySQL配置文件

MySQL的主配置文件通常在 `/etc/mysql/mysql.conf.d/mysqld.cnf` 或 `/etc/my.cnf`。你可以在这里调整MySQL的各种参数，例如字符集、缓冲区大小等。

例如，配置默认字符集为 `utf8mb4`：
```ini
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
```
修改后，需要重启MySQL服务使配置生效：
```bash
sudo systemctl restart mysql
```

### 7.2 卸载MySQL

如果需要卸载MySQL：
```bash
sudo systemctl stop mysql                  # 停止MySQL服务
sudo apt autoremove --purge mysql-server mysql-client mysql-common -y # 卸载软件包
sudo rm -rf /etc/mysql /var/lib/mysql      # 删除残留文件和数据目录
sudo rm /etc/apt/sources.list.d/mysql.list # 删除APT存储库配置
sudo dpkg -r mysql-apt-config              # 移除存储库配置包
```

至此，你已成功在Ubuntu上安装并配置了MySQL 5.7数据库。你可以通过命令行客户端或图形化工具连接到数据库，并开始你的数据库操作。
