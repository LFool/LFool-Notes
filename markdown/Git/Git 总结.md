# Git 总结



> 主要介绍 Git 的常用操作

首先本地 Git 有三个分区：`working directory`，`stage/index area`，`commit history`

`working directory`：工作目录

`stage/index area`：暂存区，`git add`命令会把`working directory`中的修改添加到暂存区

`commit history`：`git commit`命令会把`暂存区`的内容提交到该分区。每个`commit`都有一个唯一的 Hash 值

三者的关系如下图所示：

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20211224/230345164035822516403582259772Qrb6f.jpg" alt="图片" style="zoom:50%;" />

## 常用 git 命令

```bash
# 初始化 git 项目
git init
# working dir -> stage area
git add .
# stage area -> commit history
git commit -m "xxxx"
# 查看工作区提交情况
git status
# 查看 git commit 提交记录
git log
# 创建新分支
git branch xxx
# 切换新分支
git checkout xxx
# 创建并切换新分支
git checkout -b xxx
# 将 commit history 区的代码恢复到 stage
git reset
```

## 常用 git 操作

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20211225/155230164041875016404187507322DFwle.png" alt="image-20211225155230628" style="zoom: 33%;" />



**<font color=#5D21D0 size=2.5>把`stage`中的修改还原到`working dir`中</font>**

```bash
# 恢复某一个文件
git checkout [fileName]
# 恢复所有文件
git checkout .
```



**<font color=#5D21D0 size=2.5>把`commit history`区的历史提交还原到`working dir`中</font>**

```bash
git checkout HEAD .
```



**<font color=#5D21D0 size=2.5>将`commit history`区的文件还原到`stage`区</font>**

```bash
git reset [fileName]
git reset .
```



**<font color=#5D21D0 size=2.5>撤销`git commit`</font>**

```bash
git reset --soft HEAD^
# 等价
git reset --soft HEAD~1
# 撤销两次 git commit
git reset --soft HEAD~2

# 如果此时我们同时需要撤销远程的提交记录
# -f 表示强制推送到远程
git push -f
```

下面介绍一下几个参数

`--mixed`：不删除工作空间改动代码，撤销`commit`，并且撤销`git add .`操作

`--soft  `：不删除工作空间改动代码，撤销`commit`，不撤销`git add .`

`--hard` ：删除工作空间改动代码，撤销`commit`，撤销`git add .`



**<font color=#5D21D0 size=2.5>修改`git commit -m "  "`注释</font>**

```bash
git commit --amend
```

