# Linux学习笔记

---

# Linux基础

## VMware网络模式说明

安装Linux系统时，VMware提供三种网络连接模式：

1. **桥接模式**：虚拟系统可与外部系统双向通讯，但容易造成IP冲突。例如`192.168.0.xx`网段最多可容纳约254台主机，若每个物理主机都运行虚拟机，IP数量会翻倍，极易引发地址冲突。
2. **NAT模式**：通过网络地址转换解决IP冲突问题。例如本机处于`192.168.0.xx`网段，NAT模式下虚拟机会生成`192.168.100.xx`网段的IP，不会与物理网络冲突；虚拟机可以访问外部网络，但外部主机无法直接访问NAT内的虚拟机。
3. **主机模式**：独立封闭的网络环境，可自由配置，无法与外部通讯。

---

## 1 Linux目录结构

Linux采用单根目录树形结构，所有资源都挂载在根目录`/`下，根目录下的子目录有约定俗成的用途，不建议随意修改。**在Linux世界中，一切皆为文件。**

### 根目录核心子目录说明

| 目录 | 功能说明 |
|------|----------|
| `/bin`、`/usr/bin`、`/usr/local/bin` | 存放最常用的命令 |
| `/sbin` | 存放系统管理员使用的系统管理程序 |
| `/home` | 普通用户的家目录 |
| `/root` | 系统管理员（超级用户）的家目录 |
| `/lib` | 系统开机所需的核心动态链接共享库，类似Windows的DLL、Java的依赖包 |
| `/lost+found` | 系统非法关机时存放恢复文件，默认隐藏，通常为空 |
| `/etc` | 存放系统管理配置文件和子目录，例如MySQL的`my.cnf` |
| `/usr` | 用户应用程序和文件的主目录，类似Windows的`Program Files` |
| `/usr/local` | 编译源码安装软件的默认目录 |
| `/boot` | 存放Linux核心启动文件 |
| `/proc` | 虚拟文件系统，映射系统内存，可读取系统状态，随意操作可能导致系统崩溃 |
| `/srv` | 存放服务启动后的数据，不建议随意改动 |
| `/sys` | Linux 2.6内核新增的硬件与驱动相关虚拟文件系统 |
| `/tmp` | 存放临时文件 |
| `/dev` | 设备文件目录，所有硬件以文件形式存在，类似Windows设备管理器 |
| `/media` | 系统自动挂载可移动设备的目录，如U盘、光驱 |
| `/mnt` | 临时挂载其他文件系统的目录，VMware共享文件夹在`hgfs`子目录 |
| `/opt` | 大型软件安装目录，如Oracle数据库，默认为空 |
| `/var` | 存放动态增长的数据，如日志、软件运行数据 |
| `/selinux` | 安全子系统，可限制程序访问权限，支持自定义配置 |

---

## 2 vim编辑器

### 常用基础命令

**命令行模式**
- `:wq`：保存并退出
- `:q`：退出（未修改时可用）
- `:q!`：强制退出不保存
- `:set nu`：显示行号
- `:set nonu`：关闭行号显示

**普通模式**
- `yy`：复制当前行，前面加数字可指定复制行数；`p`：粘贴
- `dd`：删除当前行，前面加数字可指定删除行数
- `/关键字`：查找关键字，按`n`切换到下一个匹配项
- `gg`：跳转到文件首行
- `G`：跳转到文件末尾
- `u`：撤销上一步操作

### 进阶操作
- 翻页：`Ctrl+F` 向下翻页、`Ctrl+B` 向上翻页；`Ctrl+E` 向下滚动、`Ctrl+Y` 向上滚动
- 全局替换：`:%s/旧字符串/新字符串/g`
- 缩进：`>>` 当前行向右缩进
- 可视模式：`Ctrl+V` 进入块选择模式
- 位置调整：`zt` 将当前行移到屏幕顶端；`zb` 移到底端；`zz` 移到中间
- `gf`：打开光标处的文件名

---

## 3 基本Linux命令

### 关机与重启命令

| 命令 | 功能 |
|------|------|
| `shutdown -h now` | 立即关机 |
| `shutdown -h 1` | 1分钟后关机 |
| `shutdown -r now` | 立即重启 |
| `halt` | 关机 |
| `reboot` | 立即重启 |
| `sync` | 将内存数据同步写入磁盘 |

> **注意**：
> 1. 关机/重启前建议先执行`sync`，防止数据丢失
> 2. 现代`shutdown`/`reboot`等命令已内置sync操作，但手动执行更稳妥

### 用户管理
- `su - 用户名`：切换用户，`-`表示完整加载环境变量；高权限切低权限无需密码
- `logout`：注销当前终端登录
- `useradd 用户名`：创建新用户
- `userdel 用户名`：删除用户
- `rm -rf 文件/目录`：强制递归删除，谨慎使用
- `passwd 用户名`：修改用户密码；仅输入`passwd`修改当前用户密码
- `who am i`：查看当前登录信息
- `groupadd 组名`：新建用户组
- `groupdel 组名`：删除用户组
- `useradd -g 组名 用户名`：创建用户并加入指定组
- `usermod -g 组名 用户名`：将用户移动到指定组

> Shell是命令解释器，将用户指令翻译为内核可识别的信号，常见有bash、tcsh等，国内主流使用bash。
> 用户密码信息存于`/etc/shadow`，组信息存于`/etc/group`。

### 运行级别
Linux有7种运行级别：
- 0：关机
- 1：单用户模式（找回密码用）
- 2：多用户无网络
- 3：多用户有网络（命令行模式）
- 4：系统保留
- 5：图形界面模式
- 6：重启

常用级别为3和5：
- `systemctl get-default`：查看当前默认运行级别
- `systemctl set-default multi-user.target`：设为级别3
- `systemctl set-default graphical.target`：设为级别5

### 帮助与路径命令
- `man 命令名`：查看命令手册，按空格翻页
- `pwd`：显示当前目录绝对路径
- `cd ~`：回到当前用户家目录
- `cd ..`：返回上一级目录
- `cd ../../`：向上退回两级

### CentOS7找回root密码
1. 启动界面按`e`进入编辑模式
2. 在`LANG=zh_CN.UTF-8`后追加`init=/bin/sh`，按`Ctrl+X`进入单用户模式
3. 执行：`mount -o remount,rw /` 重新挂载根目录为可写
4. 执行`passwd`，按提示输入新密码
5. 执行`touch /.autorelabel`，再执行`exec /sbin/init`重启系统

### 文件目录操作
- `mkdir 目录名`：创建目录
- `mkdir -p 多级目录`：递归创建多级目录
- `rmdir 目录名`：删除空目录
- `rm 文件`：删除文件；`-r`递归删除目录；`-f`强制删除无提示
- `touch 文件名`：创建空文件
- `cp 源文件 目标目录`：复制文件
- `cp -r 源目录 目标目录`：递归复制目录；`\cp`跳过覆盖提示
- `mv 源 目标`：移动文件/重命名
- `cat 文件名`：查看文件内容；`-n`显示行号
- `more 文件名`：分页查看文件
- `less 文件名`：分页查看，按需加载，功能比more更强
- `echo 内容`：输出到控制台，可打印环境变量如`echo $HOSTNAME`
- `head 文件名`：显示文件头部，默认10行；`-n 行数`指定行数
- `tail 文件名`：显示文件尾部；`-f`实时监控文件更新
- `>`：覆盖输出；`>>`：追加输出
- `ln -s 目标路径 链接路径`：创建软链接（快捷方式）
- `history`：查看命令历史记录

### 时间日期类
- `date`：显示当前时间
- `date "+%Y-%m-%d %H:%M:%S"`：格式化显示时间
- `date -s "2026-8-3 17:35:25"`：设置系统时间
- `cal 年份`：显示指定年份日历

### 查找搜索类
- `find 路径 -name 文件名`：递归查找文件
- `find 路径 -user 用户名`：查找指定用户的文件
- `find 路径 -size +200M`：查找大于200M的文件
- `locate 文件名`：快速查找，需先执行`updatedb`更新数据库
- `which 命令名`：查找命令的可执行文件路径
- `grep 选项 "关键字"`：过滤匹配；`-n`显示行号，`-i`忽略大小写

### 压缩解压类
- `gzip 文件`：压缩为.gz格式
- `gunzip 文件.gz`：解压.gz文件
- `zip -r 包名 目录`：压缩为zip包；`-r`递归压缩目录
- `unzip 包名`：解压zip；`-d 目标路径`指定解压位置
- `tar`：打包压缩，常用后缀`.tar.gz`
  - `-c`：创建包
  - `-v`：显示详情
  - `-f`：指定文件名
  - `-z`：打包同时压缩
  - `-x`：解包
  - 压缩示例：`tar -zcvf myhome.tar.gz /home/`
  - 解压示例：`tar -zxvf myhome.tar.gz -C /opt/tmp`

---

## 4 Linux用户组

Linux中每个用户必属于一个组，文件权限分为**所有者、所属组、其他组**三个维度。

- `ls -ahl`：查看文件所有者、所属组和权限
- `chown 所有者 文件名`：修改文件所有者
- `chgrp 组名 文件名`：修改文件所属组
- `cat /etc/group`：查看系统所有组
- `usermod -g 组名 用户名`：修改用户所属组
- `usermod -d 目录 用户名`：修改用户家目录（用户需有目录访问权限）

---

## 5 rwx权限

每个文件/目录的权限由10位字符组成：
`[文件类型][所有者权限][所属组权限][其他用户权限]`

### 权限位说明
- **第0位**：文件类型
  - `-`：普通文件
  - `d`：目录
  - `l`：软链接
  - `c`：字符设备（鼠标、键盘）
  - `b`：块设备（硬盘）
- **第1~3位**：所有者（user）权限
- **第4~6位**：所属组（group）权限
- **第7~9位**：其他用户（other）权限

### rwx含义
#### 针对文件
- `r`：可读，可查看文件内容
- `w`：可写，可修改文件内容；**删除文件需要目录的w权限**
- `x`：可执行，可运行该文件

#### 针对目录
- `r`：可读，可列出目录内文件（ls）
- `w`：可写，可在目录内增删改文件/子目录
- `x`：可执行，可进入该目录（cd）

### 修改权限 chmod
#### 符号模式
- `u`所有者、`g`所属组、`o`其他人、`a`所有人
- `+`增加、`-`移除、`=`设置

```bash
# 所有者读写执行，组和其他读执行
chmod u=rwx,g=rx,o=rx abc.txt

# 所有者移除执行，组增加写权限
chmod u-x,g+w abc.txt

# 所有用户添加读权限
chmod a+r abc.txt
```

#### 数字模式
- `r=4`、`w=2`、`x=1`
- `rwx=7`、`rx=5`、`r=4`

```bash
# 设置权限为 rwxr-xr-x
chmod 755 abc.txt
```

### 修改所有者与所属组
- `chown 所有者 文件/目录`：修改所有者
- `chown -R 所有者:组 目录`：递归修改目录及所有子文件
- `chgrp 组名 文件/目录`：修改所属组
- `chgrp -R 组名 目录`：递归修改

---

## 6 任务调度

### crond 定时任务
按周期重复执行指定命令或脚本。

#### 基本命令
- `crontab -e`：编辑定时任务
- `crontab -l`：查看当前用户的任务
- `crontab -r`：删除当前用户所有任务

#### 时间规则
格式：`分 时 日 月 星期`

| 位置 | 含义 | 取值范围 |
|------|------|----------|
| 第1位 | 分钟 | 0-59 |
| 第2位 | 小时 | 0-23 |
| 第3位 | 日期 | 1-31 |
| 第4位 | 月份 | 1-12 |
| 第5位 | 星期 | 0-7（0和7均为周日） |

特殊符号：
- `*`：任意时间
- `,`：不连续时间，如`0 8,12,16 * * *`
- `-`：连续范围，如`0 5 * * 1-6`
- `*/n`：每隔n单位执行，如`*/10 * * * *`

示例：每分钟将/etc目录列表写入文件
```bash
*/1 * * * * ls -l /etc/ > /tmp/etc_list.txt
```

#### 脚本执行方式
编写`.sh`脚本，在crontab中指定绝对路径执行。

### at 一次性任务
执行一次性计划任务，守护进程`atd`后台调度。

#### 基本命令
- `at 时间`：创建任务，输入命令后`Ctrl+D`结束
- `atq`：查看待执行任务
- `atrm 编号`：删除指定任务

#### 时间格式
- `hh:mm`：指定当天时间，已过则次日执行
- `midnight`/`noon`/`teatime`：模糊时间
- `5pm`/`10am`：12小时制
- `04:00 2026-8-14`：指定日期
- `now + 5 minutes`：相对时间
- `today`/`tomorrow`

示例：两分钟后写入时间到日志
```bash
at now + 2 minutes
date > /root/date200.log
# 按Ctrl+D结束
```

---

## 7 Linux磁盘分区

### 分区基础
硬盘分区后挂载到文件系统目录，典型分区包括：`/`根分区、swap交换分区、`/boot`启动分区。

- `lsblk`：查看分区结构
- `lsblk -f`：查看分区文件系统、UUID、挂载点

### 硬盘命名规则
- IDE硬盘：`hd`开头，如`hda`、`hdb`；分区1-4为主/扩展分区，5+为逻辑分区
- SCSI硬盘：`sd`开头，如`sda`、`sdb`；分区规则同上，目前主流服务器均使用SCSI硬盘

### 新增硬盘挂载步骤
1. 添加SCSI硬盘
2. 分区：
```bash
fdisk /dev/sdb
# n → p → 分区号默认 → 起始扇区默认 → 结束扇区默认 → w保存
```
3. 格式化：
```bash
mkfs -t ext4 /dev/sdb1
```
4. 创建挂载点：
```bash
mkdir /newdisk
```
5. 临时挂载：
```bash
mount /dev/sdb1 /newdisk/
```
6. 卸载：
```bash
umount /dev/sdb1
```
7. 永久挂载（重启生效）：
编辑`/etc/fstab`，添加：
```
/dev/sdb1    /newdisk    ext4    defaults    0 0
```

### 磁盘使用查询
- `df -h`：查看整体磁盘使用率
- `du -h --max-depth=1 /opt`：查看指定目录各级占用

### 实用统计命令
```bash
# 统计当前目录文件数
ls -l /opt | grep "^-" | wc -l

# 统计当前目录目录数
ls -l /opt | grep "^d" | wc -l

# 递归统计所有文件数
ls -lR /opt | grep "^-" | wc -l

# 树状显示目录结构
tree /opt
# 未安装则执行：yum install -y tree
```

---

## 8 网络IP与网关

- `ifconfig`：查看网卡配置信息

### 静态IP配置
1. 编辑网卡文件：
```bash
vim /etc/sysconfig/network-scripts/ifcfg-ens33
```
2. 修改/添加配置：
```
BOOTPROTO=static
IPADDR=192.168.200.130
GATEWAY=192.168.200.2
DNS1=192.168.200.2
```
3. VMware虚拟网络编辑器中，NAT模式子网设为`192.168.200.0`
4. 重启网络：
```bash
service network restart
```

### 主机名与hosts映射
- `hostname`：查看主机名
- `vim /etc/hostname`：修改主机名，重启生效

**hosts配置**：
- Windows：编辑`C:\Windows\System32\drivers\etc\hosts`
- Linux：编辑`/etc/hosts`
- 格式：`IP地址 主机名`

### 域名解析流程
浏览器访问域名时的解析顺序：
1. 浏览器本地DNS缓存
2. 系统DNS解析器缓存
3. 本地hosts文件
4. 公网DNS服务器

- Windows查看缓存：`ipconfig /displaydns`
- Windows刷新缓存：`ipconfig /flushdns`

---

## 9 进程管理

每个运行的程序都是一个进程，拥有唯一PID（进程号），分为前台进程和后台进程。

### 查看进程 ps
- `ps -aux`：显示所有进程的CPU、内存占用等详细信息
- `ps -ef`：全格式显示，可查看父进程（PPID）

`ps -aux`关键字段：
- USER：进程所属用户
- PID：进程号
- %CPU：CPU使用率
- %MEM：内存使用率
- STAT：进程状态（S休眠、R运行）
- COMMAND：进程命令

### 终止进程
- `kill PID`：终止指定进程；`-9`强制终止
- `killall 进程名`：按名称终止进程

### 进程树 pstree
- `pstree -p`：树状显示进程及PID
- `pstree -u`：树状显示进程及所属用户

### 服务管理
服务是后台守护进程，监听端口等待请求。

#### systemctl（CentOS7主流）
```bash
# 启动/停止/重启/查看状态
systemctl start/stop/restart/status 服务名

# 查看所有服务
systemctl list-unit-files

# 设置开机自启
systemctl enable 服务名

# 查看是否开机自启
systemctl is-enabled 服务名

# 关闭开机自启
systemctl disable 服务名
```

#### 防火墙端口管理
```bash
# 永久开放端口
firewall-cmd --permanent --add-port=8080/tcp

# 永久关闭端口
firewall-cmd --permanent --remove-port=8080/tcp

# 重载生效
firewall-cmd --reload

# 查询端口状态
firewall-cmd --query-port=8080/tcp
```

### 动态监控 top
实时刷新进程状态，默认3秒更新一次。
- `top -d 秒数`：指定刷新间隔
- `top -p PID`：监控指定进程

交互命令：
- `P`：按CPU使用率排序（默认）
- `M`：按内存使用率排序
- `N`：按PID排序
- `k`：终止指定进程
- `q`：退出top

### 网络监控
- `netstat -anp`：查看所有网络连接及进程
- `netstat -anp | grep sshd`：查看指定服务网络状态
- `ping 地址`：测试网络连通性

---

## 10 RPM与YUM

### RPM包管理
RPM是Linux软件打包安装工具，后缀`.rpm`。

#### 包名格式
示例：`firefox-60.2.2-1.el7.centos.x86_64`
- 名称：firefox
- 版本：60.2.2-1
- 适用系统：CentOS7 64位
- `i686`/`i386`表示32位，`noarch`表示通用

#### 基本命令
```bash
# 查询所有已安装包
rpm -qa

# 查询是否安装
rpm -q 包名

# 查询包详细信息
rpm -qi 包名

# 查询包包含的文件
rpm -ql 包名

# 查询文件属于哪个包
rpm -qf 文件路径

# 卸载
rpm -e 包名
# 强制卸载（忽略依赖）
rpm -e --nodeps 包名

# 安装
rpm -ivh 包路径
```

### YUM包管理
基于RPM的前端包管理器，自动处理依赖，从网络下载安装。

```bash
# 查询软件包
yum list | grep 软件名

# 安装软件
yum install -y 软件名
```

---

# Linux JavaEE环境

## JDK安装
1. 创建目录：
```bash
mkdir /opt/jdk
```
2. 上传JDK压缩包到`/opt/jdk`
3. 解压：
```bash
cd /opt/jdk
tar -zxvf jdk-8u261-linux-x64.tar.gz
```
4. 移动到安装目录：
```bash
mkdir /usr/local/java
mv /opt/jdk/jdk1.8.0_261 /usr/local/java/
```
5. 配置环境变量：
```bash
vim /etc/profile
```
末尾追加：
```bash
export JAVA_HOME=/usr/local/java/jdk1.8.0_261
export PATH=$JAVA_HOME/bin:$PATH
```
6. 生效并验证：
```bash
source /etc/profile
java -version
```

## Tomcat安装
1. 创建目录并解压Tomcat
2. 启动：
```bash
cd /opt/tomcat/apache-tomcat-9.0.12/bin
./startup.sh
```
3. 开放端口：
```bash
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload
```
4. 浏览器访问`http://IP:8080`验证

## MySQL安装
1. 下载并解压安装包：
```bash
mkdir /opt/mysql
cd /opt/mysql
wget [http://dev.mysql.com/get/mysql-5.7.26-1.el7.x86_64.rpm-bundle.tar](http://dev.mysql.com/get/mysql-5.7.26-1.el7.x86_64.rpm-bundle.tar)
tar -xvf mysql-5.7.26-1.el7.x86_64.rpm-bundle.tar
```
2. 卸载系统自带mariadb：
```bash
rpm -qa | grep mari
rpm -e --nodeps mariadb-libs
```
3. 按顺序安装：
```bash
rpm -ivh mysql-community-common-5.7.26-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.26-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.26-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.26-1.el7.x86_64.rpm
```
4. 启动服务：
```bash
systemctl start mysqld.service
```
5. 获取初始密码：
```bash
grep "password" /var/log/mysqld.log
```
6. 登录并修改密码：
```sql
set global validate_password_policy=0;
set password for 'root'@'localhost' = password('Zheng123');
flush privileges;
```

---

# Linux 大数据Shell

Shell是命令行解释器，提供用户与内核交互的接口，可编写脚本实现自动化运维。

## Shell入门
创建`hello.sh`：
```bash
#!/bin/bash
echo "hello,world"
```

执行方式：
```bash
# 方式1：赋予执行权限后运行
chmod +x hello.sh
./hello.sh

# 方式2：sh命令直接执行
sh hello.sh
```

## Shell变量
分为**系统变量**和**用户自定义变量**。
- 系统变量：`$HOME`、`$PWD`、`$SHELL`、`$USER`等
- `set`：查看所有变量

### 变量操作
```bash
# 定义变量，等号两边不能有空格
A=100

# 使用变量
echo $A
echo "A=$A"

# 撤销变量
unset A

# 只读变量（不可修改、不可撤销）
readonly B=2
```