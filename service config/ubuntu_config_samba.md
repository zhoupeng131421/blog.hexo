---
title: ubuntu config samba server
date: 2019-10-15
tags: [ubuntu, samba]
categories: 服务配置
---

# 一、 系统安装
- software select选中SSH SAMBA

# 二、系统基本服务配置

## NTFS挂载
- 查看所有磁盘分区 sudo fdisk -l
![](editor:1568639922306.png)
- mount命令挂载到/mnt/HomeNas/目录各个子目录:
``` shell
sudo mount -t ntfs /dev/sda3 /mnt/HomeNas/doc -o iocharset=utf8,umask=0
sudo mount -t ntfs /dev/sda4 /mnt/HomeNas/backup -o iocharset=utf8,umask=0
sudo mount -t ntfs /dev/sda5 /mnt/HomeNas/media -o iocharset=utf8,umask=0
```
- 系统启动自动mount,
/etc/fstab中添加设置：
``` shell
/dev/sda3 /mnt/HomeNas/doc ntfs utf8,umask=0
/dev/sda4 /mnt/HomeNas/backup ntfs utf8,umask=0
/dev/sda5 /mnt/HomeNas/media ntfs utf8,umask=0
```

## Samba服务配置
- 备份 smb.conf
``` shell
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```
- 添加pubic配置： sudo vim /etc/samba/smb.conf，增加如下配置：
``` shell
[HomeNas]
   comment = Home Nas
   path = /mnt/HomeNas
   browseable = yes
   writable = yes
   public = yes
```
- smb服务重启：
`sudo /etc/init.d/samba restart`
- 增加用户管理:
```shell
sudo useradd smbUser  #添加Linux用户
sudo smbpasswd -a smbUser  #smbpasswd
```
smb.conf 增加配置:
```shell
[smbNas]
   path = /mnt/HomeNas
   available = yes
   browseable = yes
   public = no
   valid users = smbUser
   writable = yes
```
- restart server
