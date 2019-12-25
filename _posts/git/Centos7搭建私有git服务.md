---
layout:     post
title:      Centos7搭建私有git服务
subtitle:   Centos7搭建私有git服务
date:       2019-12-25
author:     Arain
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - git
---

# Centos7搭建私有git服务

## 一、安装git

1. 检查git是否已安装

   ```shell
   git version
   ```

2. 安装git

   ```shell
   yum install git
   ```

##  二、创建git用户

创建一用户服务器用户（git）用于管理git仓库，我们推荐这么做，但是这不是必须的，也可选择任意已有用户。

```shell
# 检查名为git的用户是否存在
id git
# 创建名为git的用户
useradd git
# 设置git用户的密码
passwd git
```

## 三、服务端创建git仓库

例如在“/home/git/data”目录下创建名为test的仓库：

```shell
# 创建目录
mkdir -p /home/git/data/test.git
# 初始化仓库
git init --bare /home/git/data/test.git
# git默认拒绝了push操作，需要进行设置，修改.git/config文件后面添加如下代码
cd /home/git/data/test.git
git config receive.denyCurrentBranch ignore
```

## 四、客户端SSH clone远程仓库

客户端安装git后通过ssh的默认端口（22）clone服务端（192.168.10.1）仓库：

```shell
git clone git@192.168.10.1:/home/git/data/test.git
```

如果SSH用的不是默认端口（22），则需要使用以下的命令指定端口号（假设SSH端口号是8000）：

```shell
git clone ssh://git@192.168.10.1:8000/home/git/data/test.git
```

## 五、服务端开启RSA身份认证

创建ssh配置，如果此文件夹已存在则忽略

```shell
mkdir /home/git/.ssh
touch /home/git/.ssh/authorized_keys
# 设置权限，此步骤不能省略，而且权限值也不要改，不然会报错。
chmod 700 /home/git/.ssh/
chmod 600 /home/git/.ssh/authorized_keys
```

修改服务端配置文件“/etc/ssh/sshd_config”，取消以下配置的注释：

```shell
RSAAuthentication yes
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

修改配置后重启sshd服务：

```shell
systemctl restart sshd
```

## 六、客户端创建SSH公私钥

1. 检查客户端是否存在公私钥

   查看用户目录下`.ssh`文件夹内是否存在id_rsa和id_rsa.pub两个文件，若存在则无需重新生成公私钥。

2. 生成ssh公私钥

   在客户端的git bash下执行如下命令（按实际情况替换相关参数），将会在用户的`.ssh`目录下生成id_rsa和id_rsa.pub两个文件。其中id_rsa为私钥文件，id_rsa.pub为公钥文件。

   ```shell
   ssh-keygen -t rsa -C "123456789@qq.com"
   ```

## 七、客户端上传公钥到服务端

在客户端的git bash下执行如下命令。如果命令执行时报告“fatal: unrecognized command 'cat >> .ssh/authorized_keys'“异常，说明git用户没有使用bash启动。可以先修改/etc/passwd文件。

```shell
ssh git@192.168.10.1 'cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub
```

## 八、禁止git用户登录服务器

Git用户专门用来管理仓库代码，所以不建议git用户通过ssh登录到服务器。所以一般会禁止git用户通过ssh登录服务器。修改文件/etc/passwd文件将`git:x:502:504::/home/git:/bin/bash`改为`git:x:502:504::/home/git:/bin/git-shell`。修改后git用户可以通过ssh正常使用git，但是不能登陆服务器。

