# MySQL 容易被忽略的小知识

本篇文章主要介绍一些我感觉容易被忽略的小细节，主要从「查询请求执行过程」「启动选项」「系统变量」「字符集和比较规则」四个角度展开

### <font color=#1FA774>查询请求执行过程</font>



<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220701/1405531656655553NlGNuv11.svg" alt="11" style="zoom:80%;" />

**<font color='red'>连接管理</font>**

首先，查询请求之前，客户端肯定需要和服务器连接，所以第一部分是连接管理。连接方式有三种：「TCP/IP」「命名管道或共享内存」「UNIX 域套接字」

**注意：**命令`mysql -h127.0.0.1 -uroot -p`使用的是「TCP/IP」连接方式；而命令`mysql -hlocalhost -uroot -p`使用的是「UNIX 域套接字」连接方式

- localhot (local) 不经网卡传输！这点很重要，它不受网络防火墙和网卡相关的的限制
- 127.0.0.1 是通过网卡传输，依赖网卡，并受到网络防火墙和网卡相关的限制
- 一般设置程序时本地服务用 localhost 是最好的，localhost 不会解析成 ip，也不会占用网卡、网络资源

当每个客户端连接到服务器时，服务器都会为该客户端创建一个线程专门和该客户端交互。如果客户端断开连接，服务器并不会立马销毁该线程，而是缓存起来分配给其他客户端使用，避免了频繁的创建销毁线程

**<font color='red'>解析与优化</font>**

**查询缓存：**服务器会把每次的查询和结果缓存起来，以供之后相同的查询可直接使用该结果。但是每条查询请求必须任何字符都相同 (如：空格、注释、大小写) 才可以命中缓存。如果某个表发生了修改，那么与该表相关的所有缓存都会被删除

**语法解析：**对查询语句的语法进行解析，类似于编译原理中的词法解析，语法分析，语义分析

**查询优化：**可能人写的查询求情效率没有那么高，所以 MySQL 会对语句进行优化处理 (还挺聪明的！！)

**<font color='red'>存储引擎</font>**

MySQL 把数据的存储和提取操作封装到了一个名为存储引擎的模块中，各种不同的存储引擎为上层提供统一的调用接口

#### <font color=#9933FF>启动与连接</font>

注意：以下命令都是基于「MacOS」系统

**启动服务端**

- `mysqld`：该命令不常用
- `mysqld_safe`：底层调用命令`mysqld`，优点在于当服务器经常出现错误停止时，可以帮助重启，且可以记录错误日志
- `mysql.server start`：底层调用命令`mysqld_safe`
- `mysql.server stop`：关闭服务
- `mysqld_multi`：可以在一个服务器上运行多个服务器实例

**连接服务端**

- `mysql -h主机名 -u用户名 -p密码`
- `exit`或`quit`或`\q`

#### <font color=#9933FF>关于存储引擎的操作</font>

查看当前服务器支持的存储引擎：`show engines`

查看表的创建结构：`show create table engine_demo_table\G`

创建表时指定存储引擎：

```mysql
create table if not exists `engine_demo_table`(
   `id` int
)engine=InnoDB;
```

修改表的存储引擎：

```mysql
alter table engine_demo_table engine=MyISAM;
```

### <font color=#1FA774>启动选项</font>

#### <font color=#9933FF>命令行上使用选项</font>

刚刚提到了启动服务端有：`mysqld`，如果我们现在使用`mysqld --skip-networking`启动表示禁止「TCP/IP」方式的连接。如：`mysql -h127.0.0.1 -uroot -p`

刚刚我们用命令`show engines`看到了服务器的默认存储引擎，如果我们现在使用`mysqld --default-storage-engine=MyISAM`启动表示服务器默认引擎改为「MyISAM」

**<font color='red'>总结：</font>**--启动选项1[=值1] --启动选项2[=值2] ... --启动选项n[=值n]

**<font color='red'>注意：</font>**等号两边不能有空格。错误例子：`mysqld --default-storage-engine = MyISAM` ❌

**<font color='red'>小提示：</font>**如果有些选项存在短形式，可以使用`-短形式`，且启动选项和值之间可以有空格

- 例子 ( 三者等价)：`mysqld --port=3307`、`mysqld -p3307`、`mysqld -p 3307`

#### <font color=#9933FF>配置文件中使用选项</font>

**<font color='red'>注意：命令行中的启动项优先级最高！！！</font>**

除了在命令行上使用选项，还可以在配置文件中使用选项，下面给出基于「MacOS」系统的配置文件路径 **<font color='red'>(优先级从上到下依次递增)</font>**

如果下述路径中没有对应文件，可自己创建一个

|         路径         |                          备注                          |
| :------------------: | :----------------------------------------------------: |
|    `/etc/my.cnf`     |                                                        |
| `/etc/mysql/my.cnf`  |                                                        |
| `SYSCONFDIR/my.cnf`  | 使用 CMake 构建 MySQL 时使用 SYSCONFDIR 选项制定的目录 |
| `$MYSQL_HOME/my.cnf` |            特定于服务器的选项 (仅限服务器)             |
| `default-extra-file` |              命令行指定的额外配置文件路径              |
|     `~/.my.cnf`      |                    特定于用户的选项                    |
|   `~/.mylogin.cnf`   |         特定于用户的登录路径选项 (仅限客户端)          |

**<font color='red'>配置文件格式 (优先级从上到下依次递增)</font>**

```
[server]
option1
option2 = value2

[mysqld]
(具体的启动选项...)

[mysqld_safe]
(具体的启动选项...)

[client]
(具体的启动选项...)

[mysql]
(具体的启动选项...)

[mysqladmin]
(具体的启动选项...)
```



**<font color='red'>程序对应类别和能读的组</font>**

|    程序名    |        类别         |             能读取的组              |
| :----------: | :-----------------: | :---------------------------------: |
|    mysqld    |     启动服务器      |         [mysqld]、[server]          |
| mysqld_safe  |     启动服务器      |  [mysqld]、[server]、[mysqld_safe]  |
| mysql.server |     启动服务器      | [mysqld]、[server]、[mysqld.server] |
|    mysql     |     启动客户端      |          [mysql]、[client]          |
|  mysqladmin  | 启动客户端 (不常用) |       [mysqladmin]、[client]        |
|  mysqldump   | 启动客户端 (不常用) |        [mysqldump]、[client]        |

### <font color=#1FA774>系统变量</font>

**查看系统变量：**`show variables [like 匹配模式]`

通过启动选项设置系统变量有两种方式：

- 通过命令行添加启动选择：`mysqld --default-storage-engine=MyISAM --max-connections=10`

- 通过配置文件添加启动选项：

    ```
    [server]
    default-storage-engine = MyISAM
    max-connections = 10
    ```

除了在启动 MySQL 服务时设置系统变量，还可以在服务运行过程中设置系统变量。在服务器启动时会初始化一个系统变量，作用范围为`GLOBAL`，之后每当有一个客户端连接到该服务器时，服务器会单独为该客户端分配一个作用范围为`SESSION`的系统变量。下面给出三种常用命令：

- 设置不同作用范围的系统变量：`set [GLOBAL|SESSION] 系统变量名 = 值`或`set [@@(GLOBAL|SESSION).] 系统变量名 = 值`
- 只对本客户端生效 (SESSION 级别)：`set 系统变量名 = 值`
- 查看不同作用范围的系统变量：`show [GLOBAL|SESSION] variables [like 匹配模式]`

**<font color='red'>注意：</font>**

- 并不是所有的系统变量都有`GLOBAL`和`SESSION`的作用范围
- 有些系统变量只读不可写

**<font color='red'>状态变量</font>**

与系统变量相对的还有状态变量，表示服务器运行的状态，不可修改

查看不同作用范围的状态变量：`show [GLOBAL|SESSION] status [like 匹配模式]`

### <font color=#1FA774>字符集和比较规则</font>

计算机中只有 0 和 1，那么怎么表示一个字符呢？所以就需要设定一种规则用不同的 01 组合来表示不同的字符，这就叫「编码」

不同字符都是不同的 01 组合，那么如何来比较不同字符的大小呢？所有就需要一种「比较规则」

这里给出三种常用的编码：

- ASCII：一共 128 个，用 1 字节 (8 bit) 表示
- GBK：用 1-2 字节表示
- UTF-8：用 1-4 字节表示

#### <font color=#9933FF>MySQL 中的 utf8 和 utf8mb4</font>

**utf8mb3：**阉割版的 UTF-8，只使用 1-3 个字符表示

**utf8mb4：**正宗的 UTF-8，使用 1-4 个字符表示

**<font color='red'>注意：</font>**MySQL 中，utf8 是 utf8mb3 的别名

查看字符集：`show charset`

查看比较规则：`show collation [like 匹配的模式]`

- `_ci`表示不区分大小写 (case insensitive)

#### <font color=#9933FF>各级别的字符集和比较规则</font>

**<font color='red'>服务器级别</font>**

可以在配置文件中设置这两个变量的值

`show variables like 'character_set_server'`

`show variables like 'collation_server'`

**<font color='red'>数据库级别</font>**

```mysql
# 创建数据库时设置字符集和比较规则
create database 数据库名
	[[default] character set 字符集名称]
	[[default] collate 比较规则名称];
# 修改数据库的字符集和比较规则
alter database 数据库名
	[[default] character set 字符集名称]
	[[default] collate 比较规则名称];
# 查看数据库的字符集和比较规则
use 数据库名;
show variables like 'character_set_database';
```

**<font color='red'>表级别</font>**

```mysql
# 创建表时设置字符集和比较规则
create table 表名
	[[default] character set 字符集名称]
	[[default] collate 比较规则名称];
# 修改表的字符集和比较规则
use 数据库名;
alter table 表名
	[[default] character set 字符集名称]
	[[default] collate 比较规则名称];
# 查看表的字符集和比较规则
show create table 表名\G;
```

**<font color='red'>列级别</font>**

```mysql
# 创建表时设置字符集和比较规则
create table 表名 (
    列名 字符串类型 [character set 字符集名称] [collate 比较规则名称],
    其他列...
);
# 修改列的字符集和比较规则
alter table 表名 modify 列名 字符串类型 [character set 字符集名称] [collate 比较规则名称];
# 查看列的字符集和比较规则
show create table 表名\G;
```

#### <font color=#9933FF>客户端和服务端通信过程中使用的字符集</font>

下表为通信相关的系统变量及其含义

|         系统变量         |                             描述                             |
| :----------------------: | :----------------------------------------------------------: |
|   character_set_client   |  服务器认为客户端请求是按照该系统变量指定的字符集进行编码的  |
| character_set_connection | 服务器在处理请求时，会把请求字节序列从 character_set_client 转化成 character_set_connection |
|   character_set_result   | 服务器采用该系统变量指定的字符集返回给客户端的字符串进行编码 |

**<font color='red'>客户端在初次连接服务端的时候，客户端会将默认的字符集信息与用户名、密码等信息一起发送给服务器，服务器在收到后会将 character_set_client、character_set_connection、character_set_result 这三个系统变量的值初始化成客户端默认字符集</font>**

从发送请求到接受响应的过程中发生的字符集转换如下图所示：

![16](https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220701/18180016566706801nLY9e16.svg)

如果 OS character 和 character_set_result 不一致的话，终端界面将会显示乱码！！！