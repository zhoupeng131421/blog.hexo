---
title: ubuntu config samba server
date: 2019-10-15
tags: [ubuntu, samba]
categories: 服务配置
---

# I. 系统安装
- software select选中SSH SAMBA

# II. 系统基本服务配置

## NTFS挂载
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
sudo useradd -m smbUser -d /home/smbUser -s /bin/bash #添加Linux用户
sudo passwd smbUser
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


# III. mount
## Linux system
- mount -o username=xxx,password=xxx //ip/port /xxx/xxx
- auto mount: `vim /etc/fstab`: `//ip/port /xxx/xxx cifs user=xxx,pass=xxx,uid=xxx,gid=xxx,_netdev 0 0`

## windows system
- `control panel` --> `Credential Manager` --> `Windows Credentials` --> `Add a Windows credential` --> edit the server info: `ip/host, username, password`
- 打开我的电脑，在左上角的 Computer 栏中找到 `Add a network location` or `map network driver`, 输入远程samba路径 `\\x.x.x.x\xxx` 确认即可

# IV. samba with changepassword
## prepare
- setenforce 0
- systemctl stop firewalld
- yum install -y gcc httpd samba
## config samba server
- vim /etc/samba/smb.conf
```shell
[global]
   pam password change = no
   passwd chat = **NEW*UNIX*password* %n\n *Retype*new*UNIX*password* %n\n *successfully*
   passwd program = LANG=en_US /usr/bin/passwd %u
   unix password sync = yes
   passdb backend = smbpasswd
   smb passwd file = /etc/samba/smbpasswd
```
- systemctl restart smb
- useradd xxx
- passwd xxx
- smbpasswd -a xxx
## config changepassword
- wget http://prdownloads.sourceforge.net/changepassword/changepassword-0.9.tar.gz
- tar -xvzf changepassword-0.9.tar.gz
- cd changepassword-0.9
- compile Des lib:
   - cd smbencrypt/
   - tar -xzvf libdes-4.04b.tar.gz
   - cd des/
   - make
   - cp libdes.a ../
   - cd ../../
- vim conf.h
```shell
char TMPFILE[]="/changepassword-shadow-XXXXXX";
char TMPSMBFILE[]="/changepassword-smb-XXXXXX";
char TMPSQUIDFILE[]="/changepassword-squid-XXXXXX";
```
- ./configure -enable-cgidir=/var/www/cgi-bin -enable-language=Chinese -enable-smbpasswd=/etc/samba/smbpasswd -disable-squidpasswd -enable-logo=logo.jpg
   - -enable-smbpasswd=/etc/samba/smbpasswd # 修改保存samba密码的库文件
   - -disable-squidpasswd # 禁用squid
   - -enable-cgidir # 自定义apache根目录路径
   - -disable-squidpasswd # 自定义smbpassword的密码文件路径
   - -enable-logo # 设置web根目录logo文件,此处的相对路径对应的是apache根目录
      - 也就是 samba/logo.jpg对应/var/www/cgi-bin/logo.jpg
- make && make install
## config apache
- vim /etc/httpd/conf/httpd.conf
```shell
LoadModule cgid_module modules/mod_cgid.so
AddHandler cgi-script .cgi
AddDefaultCharsetb GB2312
```
- systemctl restart httpd

## verify
- http://x.x.x.x/cgi-bin/changepassword.cgi