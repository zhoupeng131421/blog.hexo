---
title: CentOS nfs config
date: 2021-2-3
tags: [CentOS, nfs]
categories: 服务配置
---

# prepare
- yum install nfs-utils
- firewalld:
    - 
- systemctl enable --now rpcbind
- systemctl enable --now nfs

# config
- `vim /etc/exports`: `/xxx x.x.x.0(ro, sync, all_squash)`
- systemctl restart rpcbind
- systemctl start nfs-server