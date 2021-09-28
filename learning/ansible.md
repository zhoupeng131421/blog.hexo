---
title: ansibel
date: 2021-03-1
tags: [ansible]
categories: Learning
---

# ansible desc
- ansible是基于Python开发，集合了众多运维工具（puppet、chef、func、fabric）的优点，实现了批量系统配置、批量程序部署、批量运行命令等功能的自动化运维工具.

# ansible install with CentOS
- yum install epel-release-7
- yum install ansible

# prepare
- generate ssh key: ssh-keygen
- distribute ssh key: ssh-copy-id -i ~/.ssh/id_rsa.pub root@xxx