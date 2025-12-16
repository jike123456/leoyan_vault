1. ctrl+L：清除终端
2. echo :你说啥，电脑就回啥，复读机。类似print
	1. echo `作为命令进行执行`:echo `pwd`
3. date cal：时间日期；cal 2025
4. 计算器：expr 5 + 3 ： 空格是必须的
5. 玩转文字：figlet “LABEX”  figlet -f slant “LABEX”：-f指定要用字体样式；slant：斜体
6. clear：工作区东西太多
7. pwd ：print word directory打印工作目录/home/itleo
8. cd：change‘directory更改目录；~；代表主目录
	1. 参数：切换到指定目录下；不写直接是home
	2. 写法：绝对路径（以/根目录为起点进行描述）cd /home/itleo/Desktop与相对路径（以当前目录为起点）cd Desktop
	3. 特殊路径符：. .. ~
		1. .:当前目录
		2. ..: 上一级目录cd ../../..回退三级
		3. ~：home目录 ^8544eb
9. ls: 当前工作目录里有什么（默认home）
	1. 选项：-a-l-h
		1. -a：all全部文件；包括隐藏（.）的文件与文件夹 -al组合形式
		2. -l: 以list形式展示信息，更加详细。
		3. -h：正常-lh文件大小有单位
	2. 参数{linux路径}
10. mkdir linux_practice：创建目录或者说是文件夹；
	1. 选项：-p：自动创建不存在的父目录；连续多层级目录
		1. 链条创建：itleo/leo/test1
	2. 参数：要创建的文件夹的路径
11. touch hello.txt: 不存在创建空文件，在则创建时间戳
	1. 参数：路径
12. echo "hello world" > hello.txt: >写入文件里
13. cat hello.txt:显示文件内容 
	1. 参数：hello.txt
14. more:等于cat，但是可翻页。
	1. 参数：一页一页翻页。
15. touch 1.txt 2.txt 3.txt;一次多个文件；ls *.txt：所有.txt结尾的文件。
16. touch note_{1..5}.txt;ls note*
17. cp hello.txt hello_copy.txt
	1. 选项：-r 文件夹带上
	2. 参数1：被复制的文件夹
	3. 参数2：复制去的地方
18. mv hello_copy.txt ..：文件夹与文件都可以
	1. mv 参数1、参数2：mv test.txt Desktop/;改名:mv test.txt test2.txt;移动文件
	2. 参数1：被移动的文件
	3. 参数2：要移动的文件
19. rm file1.txt删除文件还有文件夹
	1. -r：删除文件夹
	2. -f：force强制删除
		1. su - root ;进入管理员模式；不用-f会导致要回答y，exit退出
	3. 参数1、参数2...;表示删除的文件以及文件夹
	4. *用于模糊匹配，通配符，test*    test开头；*test：test结尾的内容；*test*任何包含test的
20. which命令：which cd
	1. cd cp等都是.exe文件/usr/bin/cd
21. find命令：find 起始路径 -name "被查找文件名"
	1. find 起始路径 -size +-n[]:+-大于小于；n数值；[kMG]单位
22. grep：grep -n 关键字 文件路径 ^e594a2
	1. 选项 ：-n：在结果中显示匹配的行号
	2. 关键字：过滤的关键字特殊符号或者空格“”
	3. 文件路径：要过滤的文件路径grep -n "itleo" test.txt也可以是管道符的内容输入
23. wc：[-c -m -l -w] 文件路径
	1. 选项，-c，统计bytes数量
	2. 选项，-m，统计字符数量。
	3. 选项，-l，统计行数。
	4. 选项，-w，统计单词数量
	5. 参数：文件路径
24. 管道符：|，把左边命令的结果，作为右边命令的输入。
	1. cat itleo.txt | grep itleo
25. 重定向符：>和>>
	1. >:将左侧命令的结果 ，==覆盖==写入符号右侧的文件中
		1.  echo "hello linux" > test.txt
		2. ls > test.txt
	2. >>，将左侧命令结果，==追加==写入符号右侧文件中
		1.  echo "leo" >> test.txt
26. tail -f -num linux路径：可以查看文件尾部内容，跟踪文件的最终更改 ^a2e58b
	1. 选项--f：持续跟踪
		1. ctrl+c
	2. -num：表示查看尾部多少行
	3. 参数：文件路径默认10行
27. vi\vim:linux中经典的文本编辑器
	1. ![[Pasted image 20251209195241.png]] 命令模式：不能自由进行文本编辑；abcd执行的是命令
	2. 输入模式：编辑模式、插入模式
	3. 底线命令模式：以：开始，通常用于文件的保存
	4. vi/vim  文件路径 ^f3ff22
		1. 文件路径不存在，创建
		2. 文件路径存在，编辑
		3. ![[Pasted image 20251209203444.png]] ![[Pasted image 20251209203740.png]] ![[Pasted image 20251209204015.png]]
		4. ：wq退出vim![[Pasted image 20251209204319.png]]
		5. i，esc，/进入搜索模式
28. root管理员用户 ^ccf4b3
	1. 普通用户一般在home内不受限，但是出了home一般只有只读与执行权限：mkdir /test:权限不够
	2. su - 用户名： ^a0fc0d
		1. -符号可选，切换用户后加载环境变量
		2. 参数：用户名
		3. 切换后exit退回上个用户或者==ctrl+d==
	3. su - root: mkdir /test;ls /
	4. sudo 其他命令：临时的管理员命令
		1. 提前为普通用户配置sudo认证
			1. 切换root用户；执行visudo命令，会自动通过vi编辑器打开：/etc/sudoers
			2. 在文件最后添加：itleo ALL=(ALL)        NOPASSWD:ALL
				1. 最后NOPASSWD: ALL表示使用sudo命令，无需输入密码
			3. 最后通过wq退出保存
			4. 切换成普通用户
			5. 执行命令，均以root运行
29. 用户与用户组
	1. 配置多个用户或者用户组；用户可以加入用户组![[Pasted image 20251210143004.png]] 
	2. linux对用户组、用户权限全都可以控制
		1. 某文件可以对用户组权限进行控制，用户也可以
	3. 用户组管理：==只有root用户==可以执行
		1. 创建用户组：groupadd 用户组名
		2. 删除用户组：groupdel 用户组名
	4. 用户管理：也需要root
		1. 创建用户：useradd -g -d 用户名
			1. -g：指定用户组，不指定-g会创建同名组并自动加入，指定-g需要组已经存在
				1. useradd test2 -g itcast -d /home/test222
			2. -d：指定用户home路径，不指定-d；home目录默认：/home/用户名
			3. useradd test：未指定-g会创建同名组加入，未指定-d：/home/test
		2. 删除用户：userdel -r 用户名
			1. -r：删除用户的home目录；不使用-r，删除用户时，home目录保留
				1. userdel -r test ；userdel test2 、rm -rf test222
		3. 查看用户所属组：id用户组（root情况下）
			1. id test3
		4. 修改用户所属组：usermod -aG 用户组 用户名
			1. 指定用户加入用户组：usermod -aG itcast test4
	5. getent：查看当前系统有哪些用户
		1. getent passwd用户![[Pasted image 20251210150910.png]]
			1. 包含：用户名；密码；用户id；组id；描述信息（无用）：home路径；执行终端（默认bash）
		2. getent group用户组![[Pasted image 20251210151420.png]] 
			1. 包含：组名称；组认证；组id
30. 查看权限控制信息
	1. 认知信息![[Pasted image 20251210152118.png]] 
		1. 权限控制信息- ；d； l：- 文件；d文件夹；l软链接；用户权限；用户组；其他用户；r：有ls权限；x:有cd权限；w：touch权限
		2. 用户；用户组
31. chmod修改权限命令：只有文件文件夹所属组以及root用户可以修改
	1. chmod -R 权限 文件或者文件夹
		1. -R：对文件夹内的全部内容应用同样的操作
		2. 示例：chmod u=rwx,g=rx,o=x hello.txt chmod -R u=rxw,g=rx,o=x test
		3. 快捷写法：chmod 751 hello.txt![[Pasted image 20251210155114.png]] 
32. chown修改文件文件夹的用户与组；必须是root用户才能更改组
	1. chown -R 用户：用户组 文件或者文件夹
		1. -R；对文件夹全部内容应用
	2. chown root hello.txt将所属用户修改成root
	3. chown :root hello.txt；用户组
	4. chown root:itleo hello.txt
	5. chown -R root test
33. ls   ls ..
34. 上下箭头可以看我上下命令；tab自动补全；ctrl+C中断运行命令
35. ls --help:介绍常用选项；man ls：完整手册
36. ls -l :以长文本格式查看当前目录下所有可见文件的详细属性。
37. /sort：搜寻
38. apropos：查找与命令相关
39. apropos file | grep create想一个与你想做的事情相关的关键词
40. cd ../challenge2；cd /home/labex/project/challenge2
41. cp challenge2.txt ..==/challenge1/==指定父目录的challenge1目录
42. whoami:告知用户名
	1. 创建新用户：sudo adduser jack：超级管理员sudo、 /home/jack 为 jack 创建主目录
	ls /home id jack
	2. uid=5000(labex) gid=5000(labex) groups=5000(labex),27(sudo),121(ssl-cert),5002(public)；用户id5000；组名与id为labex，5000；
43. cat /etc/group | sort：`cat` 命令显示文件内容， `/etc/group` 是存储组信息的地方， `| sort` 字母顺序对输出进行排序。
44. cat /etc/group | grep -E "labex"：`grep` 是一个强大的搜索工具。此命令在组文件中搜索包含“labex”的行。
45.  # 创建一个新组并将用户添加到该组
	1. 创建新群组developers：sudo groupadd developers
	2. `usermod` 命令用于修改用户帐户。 `-aG` 选项将用户添加到附加组中。:sudo usermod -aG developers jack
	3. 验证 jack 是否已成为开发者组成员:groups jack
46. # 将用户添加到 sudo 组
	1. sudo usermod -aG sudo jack：此命令使用 `usermod` 修改用户帐户。 `-aG` 选项表示“追加到组”，因此它会将 jack 添加到 sudo 组，
	2. sudo groups jack：通过将 jack 添加到 sudo 用户组
47. # 理解和操作文件权限和所有权 ^89f4a4
	1. 我们来检查一下 /home 目录下的当前权限：ls -l /home
	2. 分析输出total 8 drwxr-xr-x 2 jack jack 4096 Jul 30 10:00 jack drwxr-xr-x 5 labex labex 4096 Jul 30 09:55 labex：第一个字符表示文件类型（ `d` 表示目录， `-` 表示普通文件）第一个字符表示文件类型（ `d` 表示目录， `-` 表示普通文件）这些字符后面的用户名是文件所有者，后面是组所有者。
	3. 我们创建一个新文件并更改其所有权：
	touch /home/labex/testfile            ls -l /home/labex/testfile                                         sudo chown jack:jack /home/labex/testfile                ls -l /home/labex/testfile
	我们使用 `chown` 将文件的所有者（用户和组）都更改为 jack。
	4. 我们来修改文件的权限：sudo chmod 750 /home/labex/testfile                                             ls -l /home/labex/testfile：`chmod` 命令用于更改文件的权限7（所有者）：读取（4）+ 写入（2）+ 执行（1）= 7
48. sudo useradd -m -d /home/jack -g dev -G labex jack：- `-m`：**非常重要**，表示自动创建用户的主目录（如果不存在）。
     `-d /path`：指定主目录的具体路径。`-g groupname`：指定用户的**主组** (Primary Group)。 `-G groupname`：指定用户的**辅助组/附加组** (Secondary Group)。