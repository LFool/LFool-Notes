# Shell

## 常用命令汇总

`man`：查看每种命令的文档

`man -k xxx`：查找与 xxx 相关的命令

`pwd`：显示当前目录

`ll file*Name`：只能匹配文件，不能匹配目录

- 使用通配符 「? 任意匹配一个字符 / * 任意匹配任意个字符」
- 基本上可以使用正则表达式

`ls -i fileName`：查看文件的 inode 编号

`ln -s [fileName] [linkName]`：创建一个软链接

```shell
➜  test ln -s a.txt alink
➜  test ll
-rw-r--r--  1 lfool  staff     5B  1 29 02:06 a.txt
lrwxr-xr-x  1 lfool  staff     5B  1 29 02:06 alink -> a.txt
```

`ln [fileName] [linkName]`：创建一个硬链接

```shell
➜  test ln a.txt aHardLink
➜  test ll
-rw-r--r--  2 lfool  staff     5B  1 29 02:06 a.txt
-rw-r--r--  2 lfool  staff     5B  1 29 02:06 aHardLink
➜  test ll -i
8986726 -rw-r--r--  2 lfool  staff     5B  1 29 02:06 a.txt
8986726 -rw-r--r--  2 lfool  staff     5B  1 29 02:06 aHardLink
```

`rm -rf path`：删除目录或文件

- -f：强制删除文件或目录
- -r：递归处理，将指定目录下的所有文件与子目录一并处理

`tree path`：查看目录结构

`file fileName`：查看文件类型

`cat / more / less`：显示文件内容（全显示/分批显示）

`head / tail fileName`：显示文件头部 / 最后几行内容

- -f：使显示内容保持活动状态，会动态更新显示内容
- -n：规定行数

`nohup xxx > output.log 2>&1 &`：后台运行，输出到文件中

`du`：显示文件或目录的大小

- -h：以 K，M，G 为单位，提高信息的可读性
- -s：仅显示总计
- [fileName] / [dirName] / None：未指定，则显示当前目录下的内容

`ps -ef | grep python`：查看 python 相关进程

`lsof -i:8888`：查看端口 8888 的占用情况

`kill -s 9 PID`：杀死进程 PID

## 命令行常用快捷操作

`Ctrl + a`：跳到本行的行首

`Ctrl + e`：则跳到页尾

`Ctrl + u`：删除当前光标前面的文字 

`Ctrl + k`：删除当前光标后面的文字

`Ctrl + y`：进行恢复

## 目录及用途

|  目录  |                             用途                             |
| :----: | :----------------------------------------------------------: |
|   /    |           虚拟目录的根目录，通常不会在这里存储文件           |
|  /bin  |            二进制目录，存放许多用户级的 GNU 工具             |
| /boot  |                    启动目录，存放启动文件                    |
|  /dev  |              设备目录，Linux 在这里创建设备节点              |
|  /etc  |                       系统配置文件目录                       |
| /home  |               主目录，Linux 在这里创建用户目录               |
|  /lib  |              库目录，存放系统和应用程序的库文件              |
| /media |             媒体文件，可移动媒体设备的常用挂载点             |
|  /mnt  |          挂载目录，另一个可移动媒体设备的常用挂载点          |
|  /opt  |          可选目录，常用于存放第三方软件包或数据文件          |
| /proc  |         进程目录，存放先用硬件以及当前进程的相关信息         |
| /root  |                      root 用户的主目录                       |
| /sbin  |          系统二进制目录，存放许多 GNU 管理员级工具           |
|  /run  |             运行目录，存放系统运作时的运行时数据             |
|  /srv  |               服务目录，存放本地服务的相关文件               |
|  /sys  |             系统目录，存放系统硬件信息的相关文件             |
|  /tmp  |        临时目录，可以在该目录中创建和删除临时工作文件        |
|  /usr  | 用户二进制目录，大量用户级别的 GUN 工具和数据文件都存储在这里 |
|  /var  |        可变目录，用以存放经常变化的文件，比如日志文件        |





## 文件或目录信息

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220129/011515164339011516433901156344SKXdJ.png" alt="image-20220129011515530" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220129/014826164339210616433921064753GcRJL.png" alt="image-20220129014826379" style="zoom: 50%;" />

文件类型（每行第一个字母）：目录（d）、文件（-）、字符型文件（c）、块设备（b）、软链接（l）

文件的权限：rwx | rwx | r-x -> 属主权限 | 属组权限 | 其他人权限 （r：读 w：写 x：执行 -：无权限）

文件的硬链接总数 [传送门](./Linux硬链接和软连接的区别与总结.html)

文件属主的用户名

文件属组的组名

文件的大小（以字节为单位）

文件上次修改时间

文件名或目录名

