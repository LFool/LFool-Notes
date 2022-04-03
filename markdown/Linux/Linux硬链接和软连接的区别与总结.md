# Linux硬链接和软连接的区别与总结

## 图示软硬链接的区别

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220129/01344916433912891643391289014XfX2c7.png" alt="Linux硬链接和软连接的区别" style="zoom: 67%;" />

## 硬链接

- 具有相同 inode 节点号的多个文件互为硬链接文件
- 删除硬链接文件或者删除源文件任意之一，文件实体并未被删除
- 只有删除了源文件和所有对应的硬链接文件，文件实体才会被删除
- 硬链接文件是文件的另一个入口
- 可以通过给文件设置硬链接文件来防止重要文件被误删
- 创建硬链接命令 ln 源文件 硬链接文件
- 硬链接文件是普通文件，可以用 rm 删除
- 对于静态文件（没有进程正在调用），当硬链接数为 0 时文件就被删除。注意：如果有进程正在调用，则无法删除或者即使文件名被删除但空间不会释放

**创建一个硬链接**

```shell
➜  test ln a.txt aHardLink
➜  test ll
-rw-r--r--  2 lfool  staff     5B  1 29 02:06 a.txt
-rw-r--r--  2 lfool  staff     5B  1 29 02:06 aHardLink
➜  test ll -i
8986726 -rw-r--r--  2 lfool  staff     5B  1 29 02:06 a.txt
8986726 -rw-r--r--  2 lfool  staff     5B  1 29 02:06 aHardLink
```

可以发现两个文件的 inode 编号相同，且大小相同，说明是同一个文件

## 软链接

- 软链接类似 windows 系统的快捷方式
- 软链接里面存放的是源文件的路径，指向源文件
- 删除源文件，软链接依然存在，但无法访问源文件内容
- 软链接失效时一般是白字红底闪烁
- 创建软链接命令 ln -s 源文件 软链接文件
- 软链接和源文件是不同的文件，文件类型也不同，inode 号也不同
- 软链接的文件类型是 「l」，可以用 rm 删除

**创建一个软连接**

```shell
➜  test ln -s a.txt alink
➜  test ll
-rw-r--r--  1 lfool  staff     5B  1 29 02:06 a.txt
lrwxr-xr-x  1 lfool  staff     5B  1 29 02:06 alink -> a.txt
# 查看文件的 inode 编号
➜  test ll -i
8986726 -rw-r--r--  1 lfool  staff     5B  1 29 02:06 a.txt
8986741 lrwxr-xr-x  1 lfool  staff     5B  1 29 02:06 alink -> a.txt
```

可以发现文件和指向文件的软链接的 inode 编号不同，说明是两个文件

## 硬链接和软链接的区别

原理上，硬链接和源文件的 inode 节点号相同，两者互为硬链接。软连接和源文件的 inode 节点号不同，进而指向的 block 也不同，软连接 block 中存放了源文件的路径名

实际上，硬链接和源文件是同一份文件，而软连接是独立的文件，类似于快捷方式，存储着源文件的位置信息便于指向

使用限制上，不能对目录创建硬链接，不能对不同文件系统创建硬链接，不能对不存在的文件创建硬链接；可以对目录创建软连接，可以跨文件系统创建软连接，可以对不存在的文件创建软连接

## 参考

1. [Linux硬链接和软连接的区别与总结](https://xzchsia.github.io/2020/03/05/linux-hard-soft-link/)
2. [硬链接和软连接（符号链接）的区别](https://www.cnblogs.com/sanjun/p/9971993.html)

