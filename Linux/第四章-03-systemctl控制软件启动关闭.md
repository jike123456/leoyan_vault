# 使用 systemctl 控制软件（服务）的启动与关闭

在现代的Linux发行版中（如 CentOS 7+, Ubuntu 15.04+, Debian 8+），`systemd` 成为了默认的系统和服务管理器。`systemctl` 则是与 `systemd` 交互、管理系统服务的核心命令行工具。

---

## 一、基本概念

### 1. `systemd` 是什么？
`systemd` 是一个系统初始化和服务管理器，它负责在系统启动时启动所有后台服务（daemons），并持续管理它们。它的设计目标是提供更快的启动速度、更好的依赖关系管理和更强大的日志功能。

### 2. 服务单元 (Service Unit)
在 `systemd` 的世界里，每一个需要管理的对象（如一个服务、一个挂载点、一个设备等）都被称为一个“单元”（Unit）。我们最常打交道的是“服务单元”（Service Unit），其配置文件通常以 `.service` 结尾，存放于 `/usr/lib/systemd/system/` 或 `/etc/systemd/system/` 目录下。

例如，Nginx服务的单元文件可能是 `nginx.service`。我们通过 `systemctl` 操作的就是这些单元。

---

## 二、核心管理命令

管理服务通常需要管理员权限，因此大部分命令都需要在前面加上 `sudo`。

### 1. 服务的生命周期管理

| 命令 | 功能描述 |
| :--- | :--- |
| `sudo systemctl start <服务名>` | **启动**一个服务。 |
| `sudo systemctl stop <服务名>` | **停止**一个正在运行的服务。 |
| `sudo systemctl restart <服务名>` | **重启**一个服务（先停止，再启动）。 |
| `sudo systemctl reload <服务名>` | **重新加载**服务的配置文件，而无需中断服务。这比`restart`更平滑。 |
| `sudo systemctl reload-or-restart <服务名>`| **优先重载**，如果服务不支持`reload`，则执行`restart`。 |

### 2. 服务的开机自启设置

| 命令 | 功能描述 |
| :--- | :--- |
| `sudo systemctl enable <服务名>` | **设置**一个服务为**开机自启**。它会创建一个符号链接，让服务在下次启动时自动运行。 |
| `sudo systemctl disable <服务名>` | **取消**一个服务的**开机自启**。 |

### 3. 查看服务状态

| 命令 | 功能描述 |
| :--- | :--- |
| `systemctl status <服务名>` | **查看**一个服务的**详细状态**。这是最常用、最重要的诊断命令。 |
| `systemctl is-active <服务名>` | 检查服务当前**是否正在运行** (active/inactive)。 |
| `systemctl is-enabled <服务名>` | 检查服务**是否被设为开机自启** (enabled/disabled)。 |
| `journalctl -u <服务名>` | 查看指定服务的**所有日志**。`-f` 参数可以实时跟踪日志。 |

---

## 三、实际操作示例：管理 Nginx 服务

假设我们已经通过 `sudo apt install nginx` 或 `sudo yum install nginx` 安装了Nginx。

1.  **检查 Nginx 的当前状态**
    ```bash
    systemctl status nginx
    ```
    输出会告诉你 Nginx 是否正在运行 (`Active: active (running)`)，它的进程ID (PID)，以及最近的几条日志。

2.  **启动 Nginx**
    如果服务没有运行，我们可以启动它。
    ```bash
    sudo systemctl start nginx
    ```

3.  **停止 Nginx**
    ```bash
    sudo systemctl stop nginx
    ```

4.  **重启 Nginx**
    当你修改了 Nginx 的主配置文件后，通常需要重启服务来使配置生效。
    ```bash
    sudo systemctl restart nginx
    ```

5.  **重新加载配置**
    如果你只是修改了 Nginx 的站点配置而没有动核心设置，使用 `reload` 会更好，因为它不会中断当前的用户连接。
    ```bash
    sudo systemctl reload nginx
    ```

6.  **设置 Nginx 开机自启**
    为了让服务器重启后 Nginx 能自动运行，我们需要 enable 它。
    ```bash
    sudo systemctl enable nginx
    ```
    执行后，你可能会看到一条消息，提示已创建符号链接。

7.  **取消 Nginx 开机自启**
    ```bash
    sudo systemctl disable nginx
    ```

## 四、命令总结速查表

| 目标 | 命令 |
| :--- | :--- |
| **启动服务** | `sudo systemctl start <服务名>` |
| **停止服务** | `sudo systemctl stop <服务名>` |
| **重启服务** | `sudo systemctl restart <服务名>` |
| **重载配置** | `sudo systemctl reload <服务名>` |
| **查看状态** | `systemctl status <服务名>` |
| **设为开机自启** | `sudo systemctl enable <服务名>` |
| **取消开机自启** | `sudo systemctl disable <服务名>` |
