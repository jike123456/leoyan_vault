# 第四章-13-Linux文件的上传和下载

在Linux系统管理和日常使用中，文件上传和下载是常见的操作。无论是将本地文件传输到远程服务器，还是从远程服务器下载文件到本地，Linux提供了多种工具和方法来完成这些任务。本章将介绍几种常用的文件传输方式。

## 1. `scp` 命令：安全复制文件

`scp` (secure copy) 是基于SSH协议进行文件传输的命令。它提供了加密的数据传输，安全可靠，是远程文件传输的首选工具之一。

**基本语法：**

*   **本地文件上传到远程服务器：**
    ```bash
    scp [选项] <本地文件路径> <远程用户>@<远程主机IP或域名>:<远程目录路径>
    ```
*   **从远程服务器下载文件到本地：**
    ```bash
    scp [选项] <远程用户>@<远程主机IP或域名>:<远程文件路径> <本地目录路径>
    ```

**常用选项：**
*   `-P <port>`: 指定远程主机的SSH端口（默认为22）。
*   `-r`: 递归复制整个目录。
*   `-i <identity_file>`: 指定用于认证的私钥文件路径。
*   `-p`: 保留源文件的修改时间、访问时间和权限。
*   `-C`: 启用压缩。

**示例：**

*   **上传文件：** 将本地的 `~/document.pdf` 上传到远程服务器 `server.example.com` 的 `/home/user/docs/` 目录。
    ```bash
    scp ~/document.pdf user@server.example.com:/home/user/docs/
    ```
*   **下载文件：** 从远程服务器下载 `/var/log/syslog` 到本地的 `/tmp/` 目录。
    ```bash
    scp user@server.example.com:/var/log/syslog /tmp/
    ```
*   **上传目录：** 将本地的 `~/myproject/` 目录上传到远程服务器的 `/opt/` 目录。
    ```bash
    scp -r ~/myproject/ user@server.example.com:/opt/
    ```
*   **指定端口下载文件：**
    ```bash
    scp -P 2222 user@server.example.com:/etc/hosts ./
    ```

## 2. `sftp` 命令：交互式安全文件传输

`sftp` (SSH File Transfer Protocol) 也是基于SSH协议的文件传输工具，它提供了一个交互式的界面，类似于传统的FTP客户端，但所有数据都是加密传输的。

**基本用法：**
```bash
sftp <远程用户>@<远程主机IP或域名>
```

**进入sftp会话后常用命令：**

*   `ls` / `lls`: 列出远程/本地目录内容。
*   `cd` / `lcd`: 切换远程/本地目录。
*   `get <远程文件>`: 从远程下载文件到本地。
*   `put <本地文件>`: 将本地文件上传到远程。
*   `mget <远程文件...>`: 下载多个远程文件。
*   `mput <本地文件...>`: 上传多个本地文件。
*   `help`: 获取帮助。
*   `exit` / `bye`: 退出sftp会话。

**示例：**
```bash
sftp user@server.example.com
sftp> ls
sftp> cd /var/www/html
sftp> get index.html
sftp> lcd ~/downloads
sftp> put myfile.txt
sftp> bye
```

## 3. `rsync` 命令：高效的远程文件同步

`rsync` 是一个功能强大的文件同步工具，它可以在本地和远程系统之间同步文件和目录。`rsync` 的主要优点是它只传输发生变化的部分，而不是整个文件，这使得它在传输大量文件或更新已有文件时非常高效。

**基本语法：**
```bash
rsync [选项] <源路径> <目标路径>
```

**常用选项：**
*   `-a`: 归档模式，等同于 `-rlptgoD`，表示递归、保留符号链接、权限、时间戳、组、所有者、设备文件和特殊文件。
*   `-v`: 详细模式，显示传输过程。
*   `-z`: 压缩文件数据进行传输。
*   `-h`: 以人类可读的格式输出。
*   `--progress`: 显示传输进度。
*   `--delete`: 删除目标路径中存在但源路径中不存在的文件（慎用）。
*   `-e ssh`: 指定通过SSH进行远程连接（rsync默认使用ssh）。

**示例：**

*   **本地目录同步到远程服务器：** 将本地 `~/data/` 目录同步到远程服务器 `/backup/` 目录。
    ```bash
    rsync -avzh --progress ~/data/ user@server.example.com:/backup/
    ```
    注意源路径 `/data/` 末尾的斜杠，它表示复制目录内的内容，而不是复制目录本身。
*   **从远程服务器同步目录到本地：**
    ```bash
    rsync -avzh --progress user@server.example.com:/var/www/html/ ./website_backup/
    ```
*   **只同步修改过的文件，并删除目标中多余的文件：**
    ```bash
    rsync -avzh --delete --progress ~/mywebsite/ user@server.example.com:/var/www/html/
    ```

## 4. `wget` 和 `curl` 命令：通过HTTP/HTTPS下载文件

虽然 `wget` 和 `curl` 主要用于网络请求和下载，它们也可以用于从HTTP/HTTPS服务器下载文件。在“第四章-08-网络请求和下载”中已详细介绍。

**示例：**
*   使用 `wget` 下载文件：
    ```bash
    wget https://example.com/software.tar.gz
    ```
*   使用 `curl` 下载文件：
    ```bash
    curl -O https://example.com/document.pdf
    ```

## 5. 基于图形界面的文件传输工具

对于桌面Linux环境或者通过SSH客户端连接到远程Linux服务器，也可以使用图形界面的文件传输工具，例如：

*   **FileZilla：** 一款流行的跨平台FTP、FTPS、SFTP客户端。
*   **WinSCP (Windows)：** Windows下的SSH客户端，集成了SCP、SFTP等功能，提供了图形化的文件管理界面。
*   **Nautilus (GNOME) / Dolphin (KDE)：** Linux桌面环境下的文件管理器通常支持通过`sftp://`协议直接连接到远程服务器进行文件浏览和传输。

选择哪种文件传输方式取决于具体的需求、网络环境以及个人偏好。对于自动化脚本或命令行操作，`scp` 和 `rsync` 是非常高效和安全的工具；对于交互式会话，`sftp` 提供了便利；而 `wget` 和 `curl` 则适用于从Web服务器下载文件。
