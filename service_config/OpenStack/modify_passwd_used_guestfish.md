---
title: OpenStack guestfish
date: 2021-9-30
tags: [OpenStack, guestfish]
categories: 服务配置
---

> libguestfs 是一组 Linux 下的 C 语言的 API ，用来访问或修改虚拟机的磁盘映像文件的工具

# install & config
- `yum install libguestfs-tools`
- `export LIBGUESTFS_BACKEND=direct`
- `wget https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-2009.qcow2`
# use
## modify default passwd for root
- generate the cryptograph of "123456"
    - `openssl passwd -1 123456`
    - $1$nZbhzwVz$cGfRsCZdzByy.FhzbnQ/W.
- `guestfish CentOS-7-x86_64-GenericCloud-2009.qcow2`
```shell
><fs> run
 100% ... ...
><fs> lish-filesystems
/dev/sda1: xfs
><fs> mount /dev/sda1 /
><fs> vi /etc/shadow    # repeated the password for root account
><fs> quit
```