[toc]

# Mac 整理

## 改变启动台图标大小

```bash
# 设置一行 10 个
defaults write com.apple.dock springboard-columns -int 10
# 设置一列 8 个
defaults write com.apple.dock springboard-rows -int 8
defaults write com.apple.dock ResetLaunchPad -bool TRUE
killall Dock
```

## 显示/关闭隐藏文件

```bash
# 显示隐藏文件
defaults write com.apple.finder AppleShowAllFiles -bool true
killall Finder

# 关闭隐藏文件
defaults write com.apple.finder AppleShowAllFiles -bool false
killall Finder
```

## vim 高亮显示行号

```bash
cp /usr/share/vim/vimrc ~/.vimrc

vim ~/.vimrc

syntax on # 主要是这一句
set nu! # 显示行数，可以不加
set autoindent # 自动缩进，加不加也可以
```

## 文件所有权限

```bash
viv
```

## 安装 zsh

```bash
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

修改命令行前缀

```bash
sudo vim /etc/zshrc

# 修改 PS1
PS1="%m %1~ %# "
```

### 安装插件

```bash
# 自动提示插件 zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

# 命令着色插件 zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

vim ~/.zshrc

# 添加 多个插件用空格分开
plugins=(zsh-autosuggestions zsh-syntax-highlighting)
```

## zsh & bash 切换

从 bash 切换到 zsh

```bash
chsh -s /bin/zsh
```

从 zsh 切换到 bash

```bash
chsh -s /bin/bash
```

zsh & bash 的环境变量

- bash 的环境变量 ./bash_profile 文件
- zsh 的环境变量 ./zshrc 文件

## clashx 设置终端代理

```bash
# 临时
echo export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890

echo export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890 >> ~/.zshrc
```

## Brew 安装 卸载 常用操作

```bash
# brew 安装脚本
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

echo export PATH=/opt/homebrew/bin:$PATH >> ~/.zshrc


# 常用命令
# 安装软件：
brew install xxx
# 卸载软件：
brew uninstall xxx
# 搜索软件：
brew search xxx
# 更新软件：
brew upgrade xxx
# 查看列表：
brew list
# 更新brew：
brew update
# 清理所有包的旧版本：
brew cleanup
# 清理指定包的旧版本：
brew cleanup $FORMULA
# 查看可清理的旧版本包，不执行实际操作：
brew cleanup -n
```

## git 设置 / 取消 代理

```bash
# 设置代理
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy https://127.0.0.1:7890

# 取消代理
git config --global --unset http.proxy
git config --global --unset https.proxy
npm config delete proxy

git config --global --list
```

## git 修改用户名 & 邮箱

```bash
git config --global user.name "LFool"
git config --global user.email "1712199884@qq.com"
```

## npm 设置 / 取消代理

```bash
# 设置代理
npm config set proxy "http://127.0.0.1:7890"
npm config set https-proxy "http://127.0.0.1:7890"

# 取消代理
npm config delete proxy
npm config delete https-proxy

# 查看所有 config
npm config list
```

## npm 修改源

```bash
npm config set registry 'https://registry.npm.taobao.org'
```

## Conda 设置

```bash
# 不自动启动 conda 环境
conda config --set auto_activate_base false
```

## 安装 MySQL

```bash
# 下载
brew install mysql@5.7

# 配置环境变量
echo 'export PATH="/opt/homebrew/Cellar/mysql@5.7/5.7.37/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# 启动服务
mysql.server start
# 初始化
mysql_secure_installation

Press y|Y for Yes, any other key for No: N # 密码长度必须8位以上（yes）
Remove anonymous users? (Press y|Y for Yes, any other key for No) : y # 移除不用密码的那个账户
Disallow root login remotely? (Press y|Y for Yes, any other key for No) : n # 不接受root远程登录账号
Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y # 删除text数据库
Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y
```

## 安装 redis

```bash
# 下载
brew install redis

# 启动 redis 服务
# 方法一：使用 brew
brew services start redis
# 方法二
redis-server /usr/local/etc/redis.conf # 以指定配置启动服务
redis-server # 以默认配置启动服务

# 使用客户端连接 redis（密码为空）
redis-cli -h 127.0.0.1 -p 6379

# 测试
127.0.0.1:6379> ping
PONG
```

## 简易安装部分

```bash
# maven
brew install maven
# node
brew install node
# java8
brew tap mdogan/zulu
brew install --cask zulu-jdk8
# miniforge
brew install --cask miniforge
```
