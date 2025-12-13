# 第五章-06-Tomcat安装部署[单机软件]

Apache Tomcat是一个开源的Java Servlet容器，用于部署和运行Java Web应用程序。本节将详细介绍在Linux系统（以CentOS或Ubuntu为例）上安装和部署Apache Tomcat的步骤。

## 1. 环境准备

*   **操作系统：** CentOS 7/8 或 Ubuntu Server 18.04/20.04 (或更高版本)
*   **权限：** `root` 用户或具有 `sudo` 权限的用户
*   **网络：** 确保服务器可以访问互联网以下载必要的软件包
*   **Java环境：** Tomcat依赖于Java运行环境（JRE或JDK）。建议安装OpenJDK。

### 1.1 安装Java开发工具包 (JDK)

**对于CentOS/RHEL：**
```bash
sudo yum install java-1.8.0-openjdk-devel -y
# 验证Java版本
java -version
```

**对于Ubuntu/Debian：**
```bash
sudo apt update
sudo apt install openjdk-8-jdk -y
# 验证Java版本
java -version
```

### 1.2 创建Tomcat用户（推荐）

出于安全考虑，不建议使用root用户运行Tomcat。我们可以创建一个专门的系统用户。
```bash
sudo groupadd tomcat
sudo useradd -s /bin/nologin -g tomcat -d /opt/tomcat tomcat
```

## 2. 下载Tomcat

访问Apache Tomcat官网（tomcat.apache.org/download-90.cgi）获取最新稳定版的Tomcat二进制包。通常选择`tar.gz`压缩包。

```bash
# 这里以Tomcat 9为例，请根据官网最新版本修改URL
wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.83/bin/apache-tomcat-9.0.83.tar.gz
```

## 3. 安装Tomcat

### 3.1 解压并移动到指定目录

建议将Tomcat安装到 `/opt` 目录下。

```bash
sudo mkdir -p /opt/tomcat
sudo tar -zxvf apache-tomcat-9.0.83.tar.gz -C /opt/tomcat --strip-components=1
```
`--strip-components=1` 会去除压缩包内的顶层目录，直接将内容解压到 `/opt/tomcat`。

### 3.2 设置目录权限

将Tomcat的安装目录权限设置为之前创建的`tomcat`用户。

```bash
sudo chown -R tomcat:tomcat /opt/tomcat/
sudo chmod -R u+rwx /opt/tomcat/conf # 确保Tomcat用户对conf目录有读写权限
sudo chmod -R g+r /opt/tomcat/conf # 确保Tomcat组对conf目录有读权限
sudo chmod -R o-rwx /opt/tomcat/conf # 移除其他用户的写权限
```
对于`logs`, `temp`, `webapps`, `work`目录，需要赋予Tomcat用户完全的读写权限。
```bash
sudo chmod -R u+rwx /opt/tomcat/logs
sudo chmod -R u+rwx /opt/tomcat/temp
sudo chmod -R u+rwx /opt/tomcat/webapps
sudo chmod -R u+rwx /opt/tomcat/work
```

## 4. 配置Tomcat服务（Systemd）

为了方便管理Tomcat，我们将其配置为一个Systemd服务。

### 4.1 创建Systemd服务文件

```bash
sudo vi /etc/systemd/system/tomcat.service
```
将以下内容粘贴到文件中：

```ini
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

Environment="JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk" # 根据实际JDK路径修改
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
Environment="TOMCAT_USER=tomcat" # Tomcat运行用户

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

User=tomcat
Group=tomcat
UMask=0007 # 确保日志文件具有合适的权限
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```
**注意：**
*   `JAVA_HOME` 变量需要根据你实际的JDK安装路径进行修改。你可以通过 `sudo update-alternatives --config java` 或 `echo $JAVA_HOME` 来查看。
*   `User=tomcat` 和 `Group=tomcat` 确保Tomcat以我们创建的用户和组运行。

### 4.2 重载Systemd并启动Tomcat

```bash
sudo systemctl daemon-reload # 重载Systemd配置
sudo systemctl start tomcat  # 启动Tomcat服务
sudo systemctl enable tomcat # 设置Tomcat开机自启
sudo systemctl status tomcat # 检查Tomcat服务状态
```
确认服务状态为 `active (running)`。

## 5. 配置防火墙

Tomcat默认使用 `8080` 端口。如果你的系统启用了防火墙，需要开放该端口。

**对于CentOS/RHEL (firewalld)：**
```bash
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

**对于Ubuntu/Debian (ufw)：**
```bash
sudo ufw allow 8080/tcp
sudo ufw enable          # 如果ufw未启用
sudo ufw status verbose
```

现在，你应该可以通过浏览器访问 `http://你的服务器IP或域名:8080` 来查看Tomcat的欢迎页面。

## 6. 配置Tomcat Manager和Host Manager (可选)

为了通过Web界面管理Tomcat（部署Web应用、管理虚拟主机等），你需要配置`manager-gui`和`admin-gui`角色，并为它们创建用户。

### 6.1 编辑 `tomcat-users.xml`

```bash
sudo vi /opt/tomcat/conf/tomcat-users.xml
```
在 `<tomcat-users>` 标签内添加以下内容：

```xml
  <role rolename="manager-gui"/>
  <role rolename="admin-gui"/>
  <user username="admin" password="YourStrongPassword" roles="manager-gui,admin-gui"/>
```
**注意：** 将 `YourStrongPassword` 替换为你的强密码。

### 6.2 允许远程访问Manager App (生产环境不推荐)

默认情况下，Tomcat Manager应用程序只允许从 `localhost` 访问。
为了从远程访问，你需要修改 `manager/META-INF/context.xml` 和 `host-manager/META-INF/context.xml` 文件。

```bash
sudo vi /opt/tomcat/webapps/manager/META-INF/context.xml
sudo vi /opt/tomcat/webapps/host-manager/META-INF/context.xml
```
找到 `<Valve>` 标签内的 `allow` 属性，将其修改为允许所有IP（不安全，仅用于测试）：
```xml
       <Valve className="org.apache.catalina.valves.RemoteAddrValve"
              allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
```
改为：
```xml
       <Valve className="org.apache.catalina.valves.RemoteAddrValve"
              allow=".*" />
```
或者允许特定IP段：
```xml
       <Valve className="org.apache.catalina.valves.RemoteAddrValve"
              allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1|192\.168\.\d+\.\d+" />
```
修改后重启Tomcat：
```bash
sudo systemctl restart tomcat
```
现在你可以通过 `http://你的服务器IP或域名:8080/manager/html` 和 `http://你的服务器IP或域名:8080/host-manager/html` 访问管理界面了。

## 7. 部署Java Web应用程序

要部署你的Java Web应用程序，只需将你的 `.war` 文件拷贝到 `/opt/tomcat/webapps/` 目录下。Tomcat会自动检测并部署。

**示例：**
```bash
sudo cp /path/to/your_app.war /opt/tomcat/webapps/
```
然后你可以通过 `http://你的服务器IP或域名:8080/your_app` 访问你的应用。

## 8. 卸载Tomcat

如果需要卸载Tomcat：
```bash
sudo systemctl stop tomcat         # 停止Tomcat服务
sudo systemctl disable tomcat      # 取消开机自启
sudo rm /etc/systemd/system/tomcat.service # 删除Systemd服务文件
sudo systemctl daemon-reload       # 重载Systemd配置
sudo rm -rf /opt/tomcat            # 删除Tomcat安装目录
sudo userdel tomcat                # 删除Tomcat用户
sudo groupdel tomcat               # 删除Tomcat组
```

至此，你已成功在Linux系统上安装并部署了Apache Tomcat，并了解了如何进行基本的管理和应用部署。
