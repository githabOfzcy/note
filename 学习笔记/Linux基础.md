# 什么是Linux

- 是一款免费、开源的操作系统，
- 完全、高效、稳定、善于处理高并发
- 创始人Linus与吉祥物企鹅tux
- 主要发行版
  - CentOSE、Redhat
  - Ubuntu
  - Suse
  - 红旗Linux
  - ……

# Linux目录结构

- 根目录
  - boot  启动Linux时使用的一些核心文件
  - lib  系统开机所需要的动态链接库
  - srv  服务启动之后需要提取的数据
  - root  系统管理员主目录 
  - sbin  存放系统管理员使用的系统管理程序
  - etc  所有的系统管理所需要的配置文件和子目录
  - usr  用户的应用程序和文件
  - bin 存放最常使用的命令
  - dev  设备管理，把所有的硬件用文件形式存储
  - media  把linux是别的设备挂载到这个目录下
  - mnt  临时挂载其他文件系统
  - sys  文件系统
  - tmp  临时文件
  - proc  系统内存的映射
  - lost+found
  - ……

- 一切皆文件

# Vi和Vim编辑器

### 三种常见模式

- 正常模式
  - 可以使用快捷键
- 插入模式/编辑模式
  - 可以输入内容
- 命令模式
  -  提供相关指令

### 三种模式相互切换

 ```mermaid
 graph TD
 a(正常模式)--->|i或者a|b(编辑模式)--->|Esc|a
 a--->|:或者/|c(命令模式)--->|Esc|a
 c-->|:wq或:q或:ql|d[退出]
 ```

### vim快捷键

##### 正常模式

1. 拷贝当前行：`yy`，粘贴：p
2. 拷贝多行：`xyy`，比如：5yy
3. 删除当前行：`dd`
4. 删除多行：`xdd`，比如：5dd
5. 查找某个单词：`/xxx`，比如：/hello，按n键为查找下一处
6. 设置行号：`：set nu`，取消行号：`:set nonu`
7. 跳转到文件首行：`gg`，跳转到文件末行：`G`
8. 撤销上一步的动作：u
9.  跳转到指定行：`x Shift+g`，比如：20 Shift+g

# Linux系统实用指令

### 运行级别：

##### 级别分类

- 存储在 /etc/inittab文件中

0. 关机

1. 单用户
2. 多用户无网络服务
3. 多用户有网络服务
4. 保留
5. 图形界面
6. 系统重启

##### 级别切换

- init [0123456]

##### 找回root密码

- 进入单用户模式，然后修改root密码
- 因为进入单用户模式，root不需要密码就可以登录
- 开机引导时 输入回车键 ，进入界面 输入e，进入新界面 选中第二行（编辑内核）输入e，文本界面 行最后输入1，回车键 输入b，进入单用户模式，使用passwd指令来修改密码

### 帮助指令

- 当对某个指令不熟悉时，使用帮助指令来了解这个指令的用法

##### 基本语法

- man [命令或配置文件]

- help [命令]

### 文件目录类

- 查看文件夹大小：du -h --max-depth=1

- 显示当前工作目录绝对路径：pwd
- 显示当前目录下的文件和文件夹：ls
  - ls -a	包括隐藏的所有文件
  - ls -l     以列表形式显示文件
  - ls -al   以列表形式显示所有文件
- 切换当前目录：cd
  - cd ~     进入根目录
  - cd /xx/xx 
  - cd ..
- 创建目录：mkdir
  - mkdir -p /xx/xxx.xx
- 删除目录：rmdir
  - rmdir xxx         删除空目录
  - rmdir -rf xxx        删除非空目录
- 创建空文件：touch
  - touch xxx.xx yyy.yy zzz.zz
- 拷贝文件：cp
  - cp xxx.xx ddd/dd/dd        拷贝文件
  - cp -r xxx yyy     拷贝整个文件夹
- 移除文件或目录：rm
  - rm -r xxx 递归删除
  - rm -f xxx 强制删除
- 移动文件目录或重命名
  - mv odlFileName newFileName
  - mv /oldPath /newPath
- 以只读方式打开文件：cat
  - cat -n xxx.txt 显示行号并查看
  - cat -n xxx.xx | more 分页显示
- 分屏查看文件内容：more与less
- 输出重定向：>
- 追加：>
- 输出内容到控制台：echo
- 显示文件开头部分：head
- 显示文件结尾部分：tail
  - tail fileName
  - tail -n5 fileName
  - tail -f filwName 实施追踪该文档的更新
- 软链接：ln
- 查看执行过的历史命令：history
  - !n : 执行指定的历史编号对应的命令

### 时间日期类

- date ：显示当前日期
  - date +%Y ：年
  - date +%m ：月 
  - date +%d ：日
  - date "+%Y-%m-%d %H:%M:%S"
- date -s "2020-1-1 11:11:11"  ：设置时间日期
- cal ：显示当前月的日历
- cal 2021：显示指定年份的日历

### 搜索查找类

- find：在指定范围内查找文件或目录
  - find ./path -name "*.text"
  - find ./path -user userName
  - find ./path -size +20M
  
- locate：利用locat数据库进行文件快速定位
  - updatedb : 建立并更新locate数据库
  - locate xxx.xx : 查找文件或目录
  
- grep：查找过滤

  - grep "aaa" *.xx ：在当前目录下查找后缀为xx的文件中包含aaa字符串的文件，并打出该字符串所在行的内容
  - grep -r "aaa" /xx ：在当前目录递归地查找文件中包含aaa的文件，并打出该字符串所在行的内容
  - grep -v "aaa" *.xx：反向查找，打印出不符合的行的内容

  - grep -n xx：显示行号
  - grep -i xx

- | ：管道符号

### 组管理和权限

- ls -ahl ：罗列文件的所有者
- groupadd xxx 添加组
- useradd -g xxx yy 在xxx组中创建yy用户
- chown userName xxx.xx 将某个文件的拥有者改为指定用户
- chgrp groupName xxx.xx 将某个文件的所在组改为指定组
- usermod -g groipName userName 更改指定用户到指定的组
- id userName 查询指定用户的信息

### 权限

ls -l后会出现像：-rw-r--r-- 1 thundersoft root  0 8月  30 10:50 test

- 第0个符号 ：文件类型
  - “ - ” ：普通文件
  - “ d ”：目录
  - " l "：软连接文件
  - " c "：字符设备，比如键盘鼠标
  - " b "：块文件，比如硬盘
- 第1-3个符号：文件所有者权限
- 第4-6个符号：文件所在组的用户的权限
- 第7-9个符号：文件其他组的用户的权限
- 第10个符号：
  - 如果是一个文件，表示硬链接的书
  - 如果是一个目录，表示该目录的子目录个数
- 后面的：拥有者、所在组、文件大小（如果是目录则为4096）、文件最后修改时间、文件名称
- rwx：文件或目录权限
  - r代表可读
  - w代表可写
  - x代表可执行
- chmod 修改权限""
  - chmod u=rwx,g=rx,o=x fileName
  - chmod o+x fileName
  - chmod a-x fileName

### 进程管理

- 在LINUX中，每一个执行的程序都称为一个进程，每个进程都分配一个ID号
- 每一个进程都会对应一个父进程，父进程可以包含多个子进程
- 前台进程与后台进程
- ps :查看进程
  - ps -a所有
  - ps -u用户
  - ps -x参数
  - ps -aux | grep xxxx 筛选包含字符“xxxx”的项
  - ps -ef 显示父进程
- kill：终止进程
  - kill num 终止对应编号的进程
  - killall name 终止对应名称的进程
  - kill -9 num 终止一个终端
- pstree：查看进程树
  - pstree -p
  - pstree -u

### Shell编程

- 
