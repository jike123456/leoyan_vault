## grep基础语法

^ebd24d

grep [选项参数] '搜索内容(模式)' [目标文件]
grep 'temporary password' /var/log/mysqld.log

## `grep` 的三种常见用法层级
1. 基础搜索 (你刚刚做的)
grep 'temporary password' /var/log/mysqld.log
2. 加上参数 (Options)
	- **`-i` (Ignore case)**：**忽略大小写**。
	    - `grep -i 'error' file.txt` -> 会把 Error, ERROR, error 全都找出来。
	        
	- **`-n` (Line number)**：**显示行号**。
	    - `grep -n 'password' file.txt` -> 告诉你密码在第几行，方便你用 `vi` 去修改。
	        
	- **`-v` (Invert match)**：**反向选择**（排除法）。
	    - `grep -v 'ok' file.txt` -> 把**不包含** "ok" 的行都显示出来（专找有问题的行）。
	        
	- **`-r` (Recursive)**：**递归搜索**。
	    - `grep -r 'password' /etc/` -> 在 `/etc` 目录及其所有子目录下查找包含 "password" 的文件。
3. 管道符配合 (The Pipe `|`)
==经常不直接读文件，而是过滤**上一个命令**的输出结果==
    ps aux | grep 'python'
	- `ps aux`：列出所有正在运行的进程（输出一大堆）。
	    
	- `|`：管道（把左边的输出传给右边）。
	    
	- `grep 'python'`：只留下包含 "python" 的行。