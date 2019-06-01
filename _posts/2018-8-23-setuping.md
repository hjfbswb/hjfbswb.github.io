---
tags: notes
modify_date: 2018-8-25
---

记录环境安装过程。

<!--more-->

## 使用ssh协议连接github 

使用ssh协议连接github，除了安全性之外有个额外的好处：不需要为每次连接提供用户名和密码。

### 检查是否已经存在SSH密钥

执行以下命令,观察~/.ssh目录下是否已经存在ssh密钥。

```sh
ls -al ~/.ssh
```
### 生成新的ssh密钥并添加到ssh-agent中

如果你还没有ssh密钥，那么需要创建新的ssh密钥。

如果你不想在每次使用ssh密钥时，都输入密钥的密码，那么你可以将ssh密钥添加到ssh-agent（负责管理ssh密钥和密钥的密码）里。

#### 生成新的ssh密钥

执行以下命令生成新的ssh密钥：

```sh
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

这个命令的执行过程中，你可以选择存放新密钥的文件路径，并且可以为新密钥设置一个安全密码。

可以使用以下命令修改已存在的ssh私有密钥的密码。

```sh
ssh-keygen -p
```

#### 将ssh密钥添加到ssh-agent中

第一步，确定ssh-agent正在运行，否则使用以下命令启动ssh-agent。

```sh
# start the ssh-agent in the background
eval $(ssh-agent -s)
```

第二步，执行以下命令将ssh私有密钥加入到ssh-agent中。

```sh
ssh-add ~/.ssh/id_rsa
```

### 将ssh公有密钥添加到github账户的配置中

### 尝试使用ssh连接github

```sh
ssh -T git@github.com
```
## jekyll安装

### 需提前安装的软件

```sh
sudo apt-get install ruby ruby-dev build-essential git libzip-dev 
```

### 安装jekyll

为了将Ruby Gems安装到当前用户下，而不是root用户，需要先修改.bashrc。
```sh
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME=$HOME/gems' >> ~/.bashrc
echo 'export PATH=$HOME/gems/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

执行以下命令安装jekyll和bundler。
```sh
gem install jekyll bundler
```

使用以下两个命令可以查看jekyll的版本。
```sh
jekyll --version
gem list jekyll
```

### 使用jekyll

在相应的目录里执行以下命令，接着就可以在浏览器里访问网站了：
```sh
bundle exec jekyll serve --host 0.0.0.0 --port 8080
```


