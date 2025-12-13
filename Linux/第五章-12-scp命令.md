# 第五章-12-scp命令

在进行集群化软件的安装和部署时，`scp` (secure copy) 命令是一个非常重要的工具，它允许你在集群中的不同主机之间安全地复制文件和目录。由于集群部署往往涉及配置文件、安装包以及日志等文件的批量传输，掌握`scp`的高效使用至关重要。

本节将侧重于`scp`在集群环境中的应用场景和一些高级用法，其基本语法和常用选项已在**第四章-13-Linux文件的上传和下载**中详细介绍，建议先阅读该章节以了解基础。

## 1. `scp` 在集群部署中的重要性

在分布式集群中，`scp`主要用于：

*   **分发安装包：** 将下载到一台机器上的软件安装包（如Hadoop、Kafka的tar包）分发到集群中的所有其他节点。
*   **同步配置文件：** 在集群的各个节点间同步相同的配置文件，确保集群配置的一致性。
*   **复制脚本和程序：** 将部署脚本、启动脚本或应用程序代码从管理节点复制到执行节点。
*   **收集日志和数据：** 从各个节点收集日志文件或生成的数据到中心存储节点进行分析。

## 2. 结合SSH免密码登录使用 `scp`

在**第五章-11-集群化软件安装前置准备**中，我们配置了SSH免密码登录。这是`scp`在集群环境中实现自动化和批处理的基础。一旦配置完成，你就可以在不输入密码的情况下，从一台机器向其他机器传输文件。

**假设 `master` 节点已配置到 `slave1` 和 `slave2` 的免密码登录。**

### 2.1 从主节点向从节点分发文件

**分发单个文件：**
```bash
scp /path/to/local/config.xml user@slave1:/path/to/remote/config.xml
scp /path/to/local/config.xml user@slave2:/path/to/remote/config.xml
```

**分发目录：**
```bash
scp -r /path/to/local/software_package/ user@slave1:/opt/
scp -r /path/to/local/software_package/ user@slave2:/opt/
```

### 2.2 从从节点向主节点收集文件

**收集单个文件：**
```bash
scp user@slave1:/path/to/remote/log.txt /path/to/local/slave1_log.txt
scp user@slave2:/path/to/remote/log.txt /path/to/local/slave2_log.txt
```

### 2.3 使用Shell脚本进行批量分发

当集群节点数量较多时，手动输入 `scp` 命令效率低下，可以使用Shell脚本自动化。

**示例：批量分发`server.properties`到所有从节点Kafka配置目录：**
```bash
#!/bin/bash

# 从节点列表 (这里假设已经在 /etc/hosts 中配置了主机名)
SLAVE_NODES=("slave1" "slave2" "slave3")
# 要分发的文件
CONFIG_FILE="/opt/kafka/config/server.properties"
# 远程目标目录
REMOTE_DIR="/opt/kafka/config/"
# 运行SCP的用户
REMOTE_USER="kafka_user"

for node in "${SLAVE_NODES[@]}"; do
    echo "Distributing $CONFIG_FILE to $node..."
    scp -p "$CONFIG_FILE" "$REMOTE_USER@$node:$REMOTE_DIR"
    if [ $? -eq 0 ]; then
        echo "Successfully distributed to $node."
    else
        echo "Failed to distribute to $node."
    fi
done
```
**注意：** `-p` 选项可以保留文件的修改时间、访问时间和权限，在集群中保持文件属性一致性很重要。

## 3. `scp` 与 `-r` 选项的注意事项

当使用 `scp -r` 复制目录时，它会复制源目录本身及其所有内容。例如：
`scp -r local_dir/ user@remote:remote_parent_dir/`
这将会在 `remote_parent_dir/` 下创建一个 `local_dir` 目录，并将内容复制进去。

如果你只想复制 `local_dir` 目录下的**内容**到 `remote_parent_dir/` 而不创建 `local_dir` 这个子目录，`scp`本身不支持像`rsync`那样在源目录末尾加 `/` 来指定。你需要做以下操作：
```bash
scp -r local_dir/* user@remote:remote_parent_dir/
```
但这只适用于 `local_dir` 下只有文件没有子目录的情况。如果 `local_dir` 下有子目录，则需要更复杂的脚本或使用 `rsync`。

## 4. `scp` 的性能考虑

*   **网络带宽：** `scp` 的传输速度受限于网络带宽。
*   **CPU消耗：** 由于数据加密解密，`scp` 会消耗一定的CPU资源。对于大量数据的传输，这可能会成为瓶颈。
*   **压缩：** 可以使用 `-C` 选项启用压缩，这在传输文本文件等可压缩数据时能提高效率，但会增加CPU开销。
    ```bash
    scp -rC /path/to/local/logs/ user@remote:/tmp/
    ```

## 5. `scp` 与 `rsync` 的选择

虽然`scp`在集群环境中非常方便，但对于需要频繁同步、增量更新或在网络状况不佳时保持传输可靠性的场景，`rsync`通常是更好的选择。`rsync`能够识别文件差异并只传输变化的部分，从而大大节省传输时间。

在集群部署初期，`scp`适合用于首次分发完整安装包和配置文件。在后续的配置更新或增量同步中，`rsync`则能发挥更大的优势。

## 6. 安全最佳实践

*   **最小权限原则：** 使用拥有所需文件访问权限的最小权限用户进行`scp`操作。
*   **限制SSH登录：** 仅允许必要的SSH免密码登录，并限制其源IP和命令执行范围。
*   **定期审查：** 定期审查`authorized_keys`文件，移除不再需要的公钥。

通过熟练运用`scp`命令，并结合SSH免密码登录和Shell脚本，你可以极大地简化集群化软件的部署和管理工作。
