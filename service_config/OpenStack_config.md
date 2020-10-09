---
title: OpenStack based config
date: 2020-10-9
tags: [CentOS, OpenStack]
categories: 服务配置
---

# 各个节点基本配置
## Prepare
- close NetworkManager
``` shell
systemctl stop NetworkManager
systemctl disable NetworkManager
```
- clse firewall
```shell
systemctl stop firewarlld
systemctl disable firewalld
```
- set hostname: `hostnamectl set-hostname xx.xx.xx`
```
network.benjo.com 192.168.1.40
```
## OpenStak package
- install yum-plugin-priorities, 防止高优先级软件被低优先级软件覆盖: `yum install -y yum-plugin-priorities`
- install epel 扩展 yum source: `yum install -y https://dl.fedoraproject.org/pub/epel/8/Everything/x86_64/Packages/e/epel-release-8-8.el8.noarch.rpm
`
- install OpenStack source: `yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-victoria/rdo-release-victoria-0.el8.noarch.rpm`
- update: `yum upgrade`
- install openstack-selinux: `yum install -y openstack-selinux`