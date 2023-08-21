---
title: PVE related configuration
date: 2022-08-09
tags: [PVE]
categories: 服务配置
---

## remove local-lvm
- `lvremove pve/data`
- `lvextend -l +100%FREE -r pve/root`
- remove local-lvm on web interface

## apt source
- `echo "deb http://download.proxmox.com/debian/pve bullseye pve-no-subscription" > /etc/apt/sources.list.d/pve-enterprise.list`
- `apt update`
- `apt upgrade`

## iommu
1. vim /etc/default/grub
```shell
#GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on"
```
- update-grub

2. vim /etc/modules
```shell
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
- update-initramfs -u -k all
- reboot

## UbuntuCloud Temp

1. create a vm with Serial and CloudInit
2. wget http://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
3. qm importdisk 100 focal-server-cloudimg-amd64.img local --format qcow2
4. resize disk
5. modify boot squence
6. convert to template

## destory cluster
1. systemctl stop pve-cluster corosync
2. pmxcfs -l
3. rm /etc/corosync/*
4. rm /etc/pve/corosync.conf
5. killall pmxcfs
6. systemctl start pve-cluster
