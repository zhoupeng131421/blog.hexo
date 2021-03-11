---
title: Change root user for ubuntu18
date: 2019-10-15
tags: [root, ubuntu]
categories: 随笔
---

# Change root user for ubuntu18
> Tencent vps 默认用户名为ubuntu，该文章旨在删除ubuntu 用户，更改为自己的root权限用户

## 1. 初始化root密码
```
sudo passwd
```
该命令用于初始化root密码

## 2. 创建用户
#### useradd 命令
```
sudo useradd -m xxx -d /home/xxx -s /bin/bash
sudo useradd -r -m -s /bin/bash xxx   #The login screen hides the user name
```
参数:
- -r 建立系统账号
- -m 自动建立用户的登录目录
- -s 指定用户登录后所使用的shell

为新用户设置密码： `sudo passwd xxx`

## 3. 修改用户权限
编辑`/etc/sudoers`文件，找到如下代码：
```
# User privilege specification
root　ALL=(ALL:ALL) ALL
```
增加sudo权限的用户名：
```
# User privilege specification
root　ALL=(ALL:ALL) ALL
XXX ALL=(ALL:ALL) ALL
```

## 4.删除默认用户`ubuntu`
新的用户`xxx`登录后，应该已经拥有root权限，在该用户下删除`ubuntu`用户名
- 删除`/etc/sudoers`文件中`ubuntu  ALL=(ALL:ALL) NOPASSWD: ALL`该行
- 然后执行`sudo userdel ubuntu`
- 删除`/home/ubuntu/`目录

## ssh root login
- sudo vim /etc/ssh/sshd-config
```shell
port=22
PermitRootLogin prohibit-password --> PermitRootLogin yes
```
- sudo service sshd restart