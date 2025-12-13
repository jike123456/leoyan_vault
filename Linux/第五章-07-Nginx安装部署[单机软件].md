# 第五章-07-Nginx安装部署[单机软件]

Nginx (发音为 "engine-x") 是一个高性能的HTTP和反向代理服务器，同时也可以作为邮件代理服务器、TCP/UDP代理服务器。因其高性能、高并发、低内存消耗等优点，Nginx在Web服务领域得到了广泛应用。本节将详细介绍在Linux系统上安装和部署Nginx的步骤。

## 1. 环境准备

*   **操作系统：** CentOS 7/8 或 Ubuntu Server 18.04/20.04 (或更高版本)
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保服务器可以访问互联网以下载必要的软件包

### 1.1 更新系统软件包并安装依赖

在安装Nginx之前，需要安装一些编译Nginx所需的依赖库。

**对于CentOS/RHEL：**
```bash
sudo yum update -y
sudo yum install epel-release -y # 安装EPEL仓库，包含Nginx包
sudo yum install nginx -y # 直接从EPEL仓库安装Nginx
# 或者如果从源码安装，需要安装以下依赖：
# sudo yum install gcc gcc-c++ make zlib-devel pcre-devel openssl-devel -y
```

**对于Ubuntu/Debian：**
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install nginx -y # 直接从官方仓库安装Nginx
# 或者如果从源码安装，需要安装以下依赖：
# sudo apt install build-essential libpcre3 libpcre3-dev zlib1g zlib1g-dev libssl-dev -y
```

## 2. 安装Nginx

### 2.1 包管理器安装 (推荐)

对于大多数用户，直接使用包管理器安装是最简单和推荐的方式。

**对于CentOS/RHEL：**
```bash
sudo yum install nginx -y
```

**对于Ubuntu/Debian：**
```bash
sudo apt install nginx -y
```

安装完成后，Nginx服务通常会自动启动并设置为开机自启。

### 2.2 源码编译安装 (高级用户)

如果你需要最新的Nginx版本，或需要自定义编译模块，可以选择源码编译安装。

1.  **下载Nginx源码包：** 访问Nginx官网（nginx.org/en/download.html）获取最新稳定版的源码包。
    ```bash
    wget http://nginx.org/download/nginx-1.24.0.tar.gz # 请替换为最新版本
    ```
2.  **解压：**
    ```bash
    tar -zxvf nginx-1.24.0.tar.gz
    cd nginx-1.24.0
    ```
3.  **配置编译选项：**
    ```bash
    ./configure \
    --prefix=/usr/local/nginx \
    --with-http_ssl_module \
    --with-http_stub_status_module \
    --with-http_realip_module \
    --with-http_gzip_static_module \
    --with-http_sub_module \
    --with-stream \
    --with-stream_ssl_module
    ```
    `--prefix` 指定安装目录，`--with-` 选项用于启用额外的模块。
4.  **编译与安装：**
    ```bash
    make
    sudo make install
    ```
    安装成功后，Nginx将安装在 `/usr/local/nginx` 目录下。

## 3. 管理Nginx服务

### 3.1 启动、停止、重启Nginx

**使用Systemctl (包管理器安装)：**
```bash
sudo systemctl start nginx    # 启动Nginx
sudo systemctl stop nginx     # 停止Nginx
sudo systemctl restart nginx  # 重启Nginx
sudo systemctl reload nginx   # 平滑重启Nginx (重新加载配置文件，不中断服务)
sudo systemctl enable nginx   # 设置开机自启
sudo systemctl disable nginx  # 取消开机自启
sudo systemctl status nginx   # 查看Nginx服务状态
```

**使用Nginx命令 (源码编译安装)：**
```bash
sudo /usr/local/nginx/sbin/nginx       # 启动Nginx
sudo /usr/local/nginx/sbin/nginx -s stop    # 停止Nginx
sudo /usr/local/nginx/sbin/nginx -s reload  # 平滑重启Nginx
```

### 3.2 检查Nginx配置文件语法

在每次修改Nginx配置文件后，都应该检查语法是否正确，以避免启动失败。
```bash
sudo nginx -t
# 或者 (源码编译安装)
sudo /usr/local/nginx/sbin/nginx -t
```
如果出现 `test is successful` 字样，则表示语法正确。

## 4. 配置防火墙

Nginx默认监听 `80` (HTTP) 和 `443` (HTTPS) 端口。如果你的系统启用了防火墙，需要开放这些端口。

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-service=http --permanent
sudo firewall-cmd --zone=public --add-service=https --permanent
sudo firewall-cmd --reload
# 或者直接开放端口
# sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
# sudo firewall-cmd --zone=public --add-port=443/tcp --permanent
# sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 'Nginx HTTP'   # 开放80端口
sudo ufw allow 'Nginx HTTPS'  # 开放443端口
# 或者更通用的 'Nginx Full' 开放80和443
# sudo ufw allow 'Nginx Full'
sudo ufw enable               # 如果ufw未启用
sudo ufw status verbose       # 检查防火墙状态
```

现在，你应该可以通过浏览器访问 `http://你的服务器IP或域名` 来查看Nginx的欢迎页面。

## 5. Nginx配置文件结构

Nginx的主要配置文件通常是 `/etc/nginx/nginx.conf` (包管理器安装) 或 `/usr/local/nginx/conf/nginx.conf` (源码安装)。

一个典型的Nginx配置文件结构如下：

```nginx
# 全局块
user  nginx; # Nginx运行用户
worker_processes  auto; # 工作进程数，auto表示根据CPU核心数自动设置

error_log  /var/log/nginx/error.log warn; # 错误日志
pid        /var/run/nginx.pid; # pid文件

events {
    worker_connections  1024; # 每个工作进程的最大连接数
}

http {
    include       /etc/nginx/mime.types; # 媒体类型映射
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main; # 访问日志

    sendfile        on; # 开启高效传输模式
    #tcp_nopush     on;

    keepalive_timeout  65; # 连接超时时间

    #gzip  on; # 开启gzip压缩

    include /etc/nginx/conf.d/*.conf; # 包含额外的配置文件 (通常用于配置虚拟主机)
}
```

## 6. 配置虚拟主机 (Server Block)

在 `/etc/nginx/conf.d/` (或 `/usr/local/nginx/conf/conf.d/`) 目录下创建新的 `.conf` 文件来配置虚拟主机。

### 6.1 配置HTTP服务

创建一个 `example.com.conf` 文件：
```bash
sudo vi /etc/nginx/conf.d/example.com.conf
```
```nginx
server {
    listen 80; # 监听80端口
    server_name example.com www.example.com; # 绑定的域名

    root /var/www/example.com; # 网站根目录
    index index.html index.htm; # 默认索引文件

    location / {
        try_files $uri $uri/ =404; # 尝试查找文件或目录，否则返回404
    }

    error_page 404 /404.html; # 自定义404页面
    location = /404.html {
        internal;
    }

    # 可选：访问日志和错误日志
    access_log /var/log/nginx/example.com_access.log main;
    error_log /var/log/nginx/example.com_error.log warn;
}
```
创建网站根目录和简单的`index.html`：
```bash
sudo mkdir -p /var/www/example.com
echo "<h1>Welcome to example.com!</h1>" | sudo tee /var/www/example.com/index.html
sudo chown -R nginx:nginx /var/www/example.com # 确保Nginx用户有权限
sudo chmod -R 755 /var/www/example.com
```

### 6.2 配置反向代理

Nginx常用于将请求转发到后端的应用服务器（如Tomcat、Node.js等）。

创建一个 `api.example.com.conf` 文件：
```bash
sudo vi /etc/nginx/conf.d/api.example.com.conf
```
```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080; # 将请求转发到本地8080端口的后端服务
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    access_log /var/log/nginx/api.example.com_access.log main;
    error_log /var/log/nginx/api.example.com_error.log warn;
}
```

修改配置文件后，务必检查语法并平滑重启Nginx：
```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 7. 卸载Nginx

如果需要卸载Nginx：

**对于CentOS/RHEL (包管理器安装)：**
```bash
sudo systemctl stop nginx
sudo yum remove nginx -y
sudo rm -rf /etc/nginx /var/log/nginx /var/cache/nginx
```

**对于Ubuntu/Debian (包管理器安装)：**
```bash
sudo systemctl stop nginx
sudo apt autoremove --purge nginx -y
sudo apt clean
sudo rm -rf /etc/nginx /var/log/nginx /var/cache/nginx
```

**对于源码编译安装：**
直接删除安装目录 `/usr/local/nginx`。
```bash
sudo rm -rf /usr/local/nginx
```

至此，你已成功在Linux系统上安装并部署了Nginx，并了解了如何进行基本的管理和配置。
