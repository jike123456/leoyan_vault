# 配置Linux固定(静态)IP地址

默认情况下，大多数Linux系统配置为通过DHCP（动态主机配置协议）==自动获取IP地址（会变的）==。这对于桌面和客户端设备很方便，但对于服务器而言，一个==固定不变的IP地址==（静态IP）是至关重要的，这样其他计算机才能稳定地找到并访问它。

本笔记将介绍在两大主流Linux发行版系列中配置静态IP的常用方法。

---

## 准备工作：获取网络信息

在开始之前，你需要从你的网络管理员或路由器设置中获取以下信息：
1.  **静态IP地址**: 你要分配给这台机器的IP地址 (例如: `192.168.1.100`)。
2.  **子网掩码**: (例如: `255.255.255.0`，等同于CIDR表示法的 `/24`)。
3.  **网关 (Gateway)**: 你的路由器地址 (例如: `192.168.1.1`)。
4.  **DNS服务器**: 用于域名解析的服务器地址 (例如: `8.8.8.8`, `114.114.114.114`)。
5.  **网络接口名称**: 你要配置的网卡名称。可以通过 `ip a` 命令查到，通常名为 `eth0`, `ens33`, `enp0s3` 等。

---

## 方法一：使用 `netplan` (适用于 Ubuntu 18.04+ 及衍生系统)

`netplan` 是一个现代化的网络配置工具，它使用简单易读的 YAML 文件来管理网络。

### 1. 找到并编辑配置文件
`netplan` 的配置文件位于 `/etc/netplan/` 目录下，通常是一个以 `.yaml` 结尾的文件，例如 `50-cloud-init.yaml` 或 `01-netcfg.yaml`。

```bash
# 查看目录下的配置文件
ls /etc/netplan/

# 使用 sudo 权限编辑该文件
sudo nano /etc/netplan/50-cloud-init.yaml
```

### 2. 编写静态IP配置
你需要修改文件内容，禁用dhcp并填入你的静态IP信息。

**重要提示**：YAML 格式对缩进非常敏感，必须使用空格，不能使用Tab键。层级之间通常是两个空格。

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    # 将 enp0s3 替换为你的实际网卡名
    enp0s3:
      dhcp4: no
      addresses:
        # IP地址和子网掩码 (CIDR格式)
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1 # 你的网关地址
      nameservers:
        # DNS服务器地址
        addresses: [8.8.8.8, 114.114.114.114]
```

### 3. 应用配置
`netplan` 提供了一个安全的测试命令，如果在120秒内没有确认，配置会自动回滚，防止因配置错误导致SSH断开连接。

```bash
# 测试配置，强烈建议在远程服务器上使用
sudo netplan try

# 如果测试无误，或者在本地机器上，可以直接应用
sudo netplan apply
```

### 4. 验证配置
使用 `ip a` 命令查看你的网卡，确认IP地址是否已经变成你所设置的静态IP。
```bash
ip a show enp0s3
```

---

## 方法二：==编辑 `ifcfg` 文件 (适用于 CentOS/RHEL 系统)==

CentOS/RHEL 系列系统长期以来使用 `/etc/sysconfig/network-scripts/` 目录下的 `ifcfg` 文件来管理网络。

### 1. 找到并编辑配置文件
文件名通常是 `ifcfg-<你的网卡名>`，例如 `ifcfg-ens33`。

```bash
# 导航到配置目录
cd /etc/sysconfig/network-scripts/

# 编辑对应的配置文件
sudo vi ifcfg-ens33
```

### 2. 编写静态IP配置
修改文件中的参数。如果某些参数不存在，请手动添加。

```ini
# 文件类型
TYPE="Ethernet"
# 启动协议，从 dhcp 改为 static
BOOTPROTO="static"
# 接口名称，与文件名和实际网卡名保持一致
NAME="ens33"
DEVICE="ens33"
# 开机时是否激活
ONBOOT="yes"

# --- 以下是需要你手动配置的核心参数 ---

# 静态IP地址
IPADDR="192.168.1.101"
# 子网掩码
NETMASK="255.255.255.0"
# 网关地址
GATEWAY="192.168.1.1"
# DNS服务器
DNS1="8.8.8.8"
DNS2="114.114.114.114"
```

### 3. 重启网络服务
修改配置后，需要重启网络服务来使其生效。

```bash
# 重启 NetworkManager 服务，这是推荐的方式
sudo systemctl restart NetworkManager
```
**警告**：在远程服务器上执行此命令有风险，如果配置错误，你将失去SSH连接。

### 4. 验证配置
同样使用 `ip a` 命令来检查IP地址是否已更新。
```bash
ip a show ens33
```
也可以尝试 `ping` 网关或外部地址来测试网络连通性。
```bash
ping 192.168.1.1
ping baidu.com
```

## 总结

配置静态IP是服务器管理的必备技能。请务必根据你所使用的Linux发行版选择正确的配置方法：
- **Ubuntu 18.04+**: 使用 `netplan` 和 YAML 文件。
- **CentOS/RHEL**: 传统上使用 `ifcfg` 脚本，但也可以使用 `nmcli` 等更现代的工具。

在任何操作前，请确保你的网络信息准确无误，并在远程操作时格外小心。
