---
title: CentOS udpate source
date: 2021-10-10
tags: [yum, centos]
categories: 随笔
---

- 建议先备份 /etc/yum.repos.d/ 内的文件（CentOS 7 及之前为 CentOS-Base.repo，CentOS 8 为CentOS-Linux-*.repo）
- 然后编辑 /etc/yum.repos.d/ 中的相应文件，在 mirrorlist= 开头行前面加 # 注释掉；并将 baseurl= 开头行取消注释（如果被注释的话），把该行内的域名（例如mirror.centos.org）替换为 mirrors.tuna.tsinghua.edu.cn。
```shell    
sudo sed -e 's|^mirrorlist=|#mirrorlist=|g' \
-e 's|^#baseurl=http://mirror.centos.org|baseurl=https://mirrors.tuna.tsinghua.edu.cn|g' \
-i.bak \
/etc/yum.repos.d/CentOS-*.repo
```
- 注意其中的*通配符，如果只需要替换一些文件中的源，请自行增删。
- 注意，如果需要启用其中一些 repo，需要将其中的 enabled=0 改为 enabled=1。
- 最后，更新软件包缓存: `sudo yum makecache`