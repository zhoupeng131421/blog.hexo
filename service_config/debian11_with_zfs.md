---
title: debian zfs
date: 2022-05-08
tags: [zfs, debian]
categories: 服务配置
---

- https://wiki.debian.org/ZFS

# install
- disable secure boot

- apt update
- vim /etc/apt/source.list
    - deb http://deb.debian.org/debian bullseye-backports main contrib non-free
- apt update
- apt install linux-headers-amd64
- apt install -t bullseye-backports zfsutils-linux

# zpool manager
- zpool create Rz01-01 sdc sdd sde sdf
- zpool add Rz01-01 cache sdb
- zfs create -V 30GB Rz01-01/pve-iscsi
- ll /dev/zvol/Rz1-01/
    - pve-iscsi -> ../../zd0
- zfs destory Rz01-01/pve-iscsi