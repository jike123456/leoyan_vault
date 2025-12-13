linux系统很==多软件（内置第三方）==均支持使用systemctl命令控制：启动、停止、开机自启

能够被systemctl管理的软件，一般称之为：**服务**

==语法==： systemctl start | stop |status | enable | disable 服务名
- start ：启动
- stop：关闭
- status：查看状态
- enable：开启开机自启

系统内置的服务比较多，比如：
- NetworkManager,主网络服务
- network,副网络服务
- firewalld,防火墙服务
- sshd,ssh服务(Finalshell远程登录Linux使用的就是这个服务

除了内置服务外，第三方软件安装后也可以以systemctl进行控制

- yum install -y ntp
可以通过ntpd服务名，配合systemctl进行控制。

- yum install -y httpd，安装apache服务器软件
可以通过httpd服务名，配合systemctl进行控制

==部分软件安装完后没有集成到systemctl中，可以进行手动添加==
