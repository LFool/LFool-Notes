[toc]

# Centos 环境配置总结

### <font color=#1FA774>命令行颜色设置</font>

```bash
vim ~/.bashrc

# 添加这一行
PS1="\[\033[1;32;1m\][\[\033[0;32;1m\]\u@\h \[\033[1;35;1m\]\W\[\033[1;32;1m\]]\[\033[1;31;1m\]\$ \[\033[1;37;1m\]"

source ~/.bashrc
```

### <font color=#1FA774>设置 vim 格式</font>

```bash
vim ~/.vimrc

set number
set tabstop=4
set shiftwidth=4
set smarttab
set cindent
set nobackup
set noswapfile
set mouse=a
colo torte
syntax on

```

### <font color=#1FA774>安装 python3</font>

```bash
# 安装依赖包
yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gcc make

yum install libffi-devel -y

# 下载 python 安装包
wget https://www.python.org/ftp/python/3.9.0/Python-3.9.0.tgz

# 解压安装
tar -zxvf Python-3.9.0.tgz
cd Python-3.9.0
./configure --prefix=/usr/local/bin/python3
make && make install

# 配置环境变量（替换系统原有的 python2 环境）
mv /usr/bin/python /usr/bin/python.bak
ln -s /usr/local/bin/python3/bin/python3 /usr/bin/python
mv /usr/bin/pip /usr/bin/pip.bak
ln -s /usr/local/bin/python3/bin/pip3 /usr/bin/pip

# 验证
python
pip -V

# 修改 yum 相关依赖
vim /usr/libexec/urlgrabber-ext-down
vim /usr/bin/yum

#!/usr/bin/python  ->  #!/usr/bin/python2.7
```

### <font color=#1FA774>安装 jdk1.8</font>

```bash
# 检查系统有没有自带open-jdk
rpm -qa |grep java
rpm -qa |grep jdk
rpm -qa |grep gcj

# 首先检索包含 java 的列表
yum list java*

# 检索 1.8 的列表
yum list java-1.8*   

# 安装 1.8.0 的所有文件
yum install java-1.8.0-openjdk* -y

# 使用命令检查是否安装成功
java -version
```

设置`JAVA_HOME`环境变量

```bash
vim /etc/profile.d/java.sh
JAVA_HOME="/usr/lib/jvm/java-1.8.0-openjdk"
source /etc/profile.d/java.sh
echo $JAVA_HOME
```

### <font color=#1FA774>安装 mysql（Centos7）</font>

```bash
yum install mysql*

yum install mariadb-server

systemctl start mariadb.service

mysqladmin -u root password xxxx
```

### <font color=#1FA774>安装 mysql（Centos8）</font>

禁用 MySQL 默认的 AppStream 存储库：

```bash
sudo dnf remove @mysql
sudo dnf module reset mysql && sudo dnf module disable mysql
```

Centos8 没有 MySQL 存储库，因此我们将使用  Centos 7 存储库。创建一个新的存储库文件。

```bash
sudo vim /etc/yum.repos.d/mysql-community.repo
```

将以下数据插入上面的存储库中

```bash
[mysql57-community]
name=MySQL 5.7 Community Server
baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/7/$basearch/
enabled=1
gpgcheck=0

[mysql-connectors-community]
name=MySQL Connectors Community
baseurl=http://repo.mysql.com/yum/mysql-connectors-community/el/7/$basearch/
enabled=1
gpgcheck=0

[mysql-tools-community]
name=MySQL Tools Community
baseurl=http://repo.mysql.com/yum/mysql-tools-community/el/7/$basearch/
enabled=1
gpgcheck=0
```

安装MySQL(这里我选择MySQL5.7)

```bash
sudo dnf --enablerepo=mysql57-community install mysql-community-server
```

 启动MySQL

```bash
systemctl start mysqld
```

查看启动状态

```bash
systemctl status mysqld
```

设置开机启动

```bash
systemctl enable mysqld
```

刷新所有修改过的配置文件

```bash
systemctl daemon-reload
```

获取安装mysql后生成的临时密码，用于登录

```bash
grep 'temporary password' /var/log/mysqld.log
```

修改登录密码

```bash
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Zz19990309!';
（修改后的密码，注意必须包含大小写字母数字以及特殊字符并且长度不能少于8位,否则会报错）
```

设置默认编码为utf-8（mysql安装后默认不支持中文）

```bash
vim /etc/my.cnf
# 进入文件后添加下面的配置即可
[mysqld]
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```

 重启MySQL服务并进入MySQL

```bash
systemctl restart mysqld
```

### <font color=#1FA774>Navicat连接阿里云服务器上MySQL数据库</font>

```mysql
use mysql;
mysql>GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'Zz19990309!' WITH GRANT OPTION;
Query OK, 0 rows affected (0.00 sec)  

mysql>FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)


# 删除授权
drop user root@'%';
FLUSH PRIVILEGES;
```

### <font color=#1FA774>CentOS8 安装配置 Redis6.2.1</font>

**下载 [安装包](https://download.redis.io/releases/redis-6.2.1.tar.gz)**

**解压**

```bash
tar -zxf redis-6.2.1.tar.gz
```

**编译**

```bash
make
make PREFIX=/usr/local/redis install
cp redis.conf /usr/local/redis/
# 注意：这里redis被安装在目录/usr/local/redis下面，redis.conf我也拷贝到这个目录下面了，如果安装目录和配置文件目录不一样的话，下面做redis.service配置文件的时候调用路径会不一样。
```

**配置 Redis 服务和开机启动**

1. 修改 redis.conf 配置文件中的两项

   ```bash
   daemonize yes # 以后台守护进程方式启动
   supervised systemd # 可以跟 systemd 进程进行交互
   ```

2. 创建配置文件：/usr/lib/systemd/system/redis.service

   ```bash
   # example systemd service unit file for redis-server
   #
   # In order to use this as a template for providing a redis service in your
   # environment, _at the very least_ make sure to adapt the redis configuration
   # file you intend to use as needed (make sure to set "supervised systemd"), and
   # to set sane TimeoutStartSec and TimeoutStopSec property values in the unit"s
   # "[Service]" section to fit your needs.
   #
   # Some properties, such as User= and Group=, are highly desirable for virtually
   # all deployments of redis, but cannot be provided in a manner that fits all
   # expectable environments. Some of these properties have been commented out in
   # this example service unit file, but you are highly encouraged to set them to
   # fit your needs.
   #
   # Please refer to systemd.unit(5), systemd.service(5), and systemd.exec(5) for
   # more information.
   [Unit]
   Description=Redis data structure server
   Documentation=documentation
   # Before=your_application.service another_example_application.service
   # AssertPathExists=/var/lib/redis
   [Service]
   # ExecStart=/usr/local/bin/redis-server --supervised systemd --daemonize yes
   # Alternatively, have redis-server load a configuration file:
   ExecStart=/usr/local/bin/redis-server /usr/local/redis/redis.conf
   ExecStop=/usr/local/bin/redis-cli shutdown
   Restart=always
   LimitNOFILE=10032
   NoNewPrivileges=yes
   # OOMScoreAdjust=-900
   # PrivateTmp=yes
   # Type=notify
   # 注意 notify 会失败，换成 forking 方式启动，让主进程复制一个子进程的方式执行
   Type=forking
   # TimeoutStartSec=100
   # TimeoutStopSec=100
   UMask=0077
   # User=root
   # Group=root
   # WorkingDirectory=/var/lib/redis
   [Install]
   WantedBy=multi-user.target
   ```

3. 执行指令

   ```bash
   systemctl enable redis
   ```

**测试验证**

```bash
systemctl start redis
redis-cli -h 127.0.0.1 -p 6379
systemctl stop redis
redis-cli -h 127.0.0.1 -p 6379
```

**允许远程连接**

```bash
vim /usr/local/redis/redis.conf

# 注释下面一行内容
# bind 127.0.0.1 -::1
```

**设置密码**

```bash
vim /usr/local/redis/redis.conf

# 去掉下面一行注释
requirepass foobared (改为自己的密码)
```

### <font color=#1FA774>CentOS8(7)更换yum源为阿里源</font>


```bash
# 1. 首先备份当前配置文件
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

# 2. 下载新的 CentOS-Base.repo 到 /etc/yum.repos.d
# 对于CentOS8
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo

# 对于CentOS7
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# 3. 运行 yum makecache 生成缓存
yum makecache
```

### <font color=#1FA774>Centos8 配置静态 ip</font>

```bash
# vim /etc/sysconfig/network-scripts/ifcfg-ens33

TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens33"
DEVICE="ens33"
ONBOOT="yes"
IPADDR="192.168.208.123"
NETMASK="255.255.255.0"
GATEWAY="192.168.208.2"
DNS1="114.114.114.114"
DNS2="8.8.8.8"
PEERDNS="yes"
PREFIX="24"


# 重新加载刚刚配置好的静态网络
nmcli c reload
```

### <font color=#1FA774>防火墙端口相关</font>

```bash
# 查看防火墙某个端口是否开放
firewall-cmd --query-port=3306/tcp
# 开放防火墙端口3306
firewall-cmd --zone=public --add-port=3306/tcp --permanent
# 开放某个范围内的端口 如10000-11000
firewall-cmd --zone=public --add-port=99-19999/tcp --permanent
# 注意：开放端口后要重启防火墙生效
firewall-cmd --reload

# 查看已经开放的端口
firewall-cmd --list-all

# 关闭防火墙端口
firewall-cmd --remove-port=3306/tcp --permanent
# 查看防火墙状态
systemctl status firewalld
# 关闭防火墙
systemctl stop firewalld
# 打开防火墙
systemctl start firewalld
# 开放一段端口
firewall-cmd --zone=public --add-port=40000-45000/tcp --permanent
# 查看开放的端口列表
firewall-cmd --zone=public --list-ports
# 查看被监听(Listen)的端口
netstat -lntp
# 检查端口被哪个进程占用
netstat -lnp|grep 3306
```

### <font color=#1FA774>安装 vim</font>

```bash
yum -y install vim*
```

### <font color=#1FA774>Centos8 安装 GCC 环境</font>

```bash
sudo dnf group install "Development Tools"
sudo dnf install man-pages
gcc --version
```

### <font color=#1FA774>Centos8 时间同步</font>

```bash
# 安装 chrony
yum install -y chrony

# 修改配置
vim /etc/chrony.conf
# pool 2.centos.pool.ntp.org iburst
server 210.72.145.44 iburst
server ntp.aliyun.com iburst

# 重新加载配置
systemctl restart chronyd.service

# 时间同步
chronyc sources -v
```

### <font color=#1FA774>通过 xshell 直接拖文件进去</font>

```bash
yum -y install lrzsz
```

### <font color=#1FA774>xshell 连接 Google cloud</font>

```bash
sudo -i 
vi /etc/ssh/sshd_config

# Authentication:
PermitRootLogin yes //默认为no，需要开启root用户访问改为yes

# Change to no to disable tunnelled clear text passwords
PasswordAuthentication yes //默认为no，改为yes开启密码登陆

passwd root

# 重启
/etc/init.d/ssh restart
```

### <font color=#1FA774>Centos8 安装 net 工具</font>

```bash
dnf -y install net-tools 
```

### <font color=#1FA774>Nginx 常用命令</font>

```bash
# 使用 nginx 操作命令前提条件：必须进入 nginx 的目录中 /usr/local/nginx/sbin

# 查看 nginx 的版本
./nginx -v

# 启动 nginx
./nginx
./nginx -c /usr/local/openresty/nginx/conf/nginx.conf

# 关闭 nginx
./nginx -s stop

# 重新加载 nginx
./nginx -s reload 

```

### <font color=#1FA774>Git 常用命令</font>

```bash
# 提交到工作区
git commit

# 查看当前分支的工作区状态（提交情况）
git status

# 创建新分支（并没有切换分支）
git branch [branch name]

# 切换分支
git checkout [branch name]

# 创建的同时切换分支
git checkout -b [branch name]

# 推送分支到远程（前提：远程无该分支）
git push -u origin [branch name]

# 合并分支（把 name 分支合并到当前分支上）
git merge [branch name]

# 另一种合并方法（垂直合并）
git rebase [name_object] # 把当前分支移动到 name_object 下

# 改变 HEAD 指向
# 一般来说 HEAD 指向分支名
# 可以改变 HEAD 指向结点
git checkout [node name]

# 获取 node name
git log
# 相对引用
git checkout [branch name | node name]^ # 指向 branch name 父结点
git checkout [branch name | node name]^^ # 指向 branch name 第二个父结点
git checkout [branch name | node name]~2 # 从 branch name 向前移动两个

# 移动分支指向某个结点
git branch -f main HEAD~3 # 把 main 分支向前移动了三个
git branch -f main HEAD # 把 main 分支移动到 HEAD 处

# 撤销
git reset # 只有本地有效
git revert # 远程也有效

# 整理提交记录
git cherry-pick [node name ...] # 把选择的 node 复制到当前分支下

git clone -b dev_jk http://10.1.1.11/service/tmall-service.git
```

### <font color=#1FA774>删除所有下载失败的 Maven jar 包</font>

```bash
for /r %i in (*.lastUpdate) do del %i
```

