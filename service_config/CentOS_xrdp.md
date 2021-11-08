---
title: CentOS7 config xrdp
date: 2020-10-9
tags: [xrdp, centos]
categories: 服务配置
---

- `sudo yum install epel-release -y`
- `sudo yum install xrdp -y`
- `sudo yum install tigervnc-server -y`
- `sudo vim /etc/xrdp/xrdp.ini`: map_bpp=32 --> 24
- selinux:
    - `sudo chcon -t bin_t /usr/sbin/xrdp`
    - `sudo chcon -t bin_t /usr/sbin/xrdp-sesman`
- systemd:
    - `sudo systemctl start xrdp`
    - `sudo systemctl enable xrdp`
- firewalld:
    - `sudo firewall-cmd  --permanent --zone=public --add-port=3389/tcp`
    - `sudo firewall-cmd --reload`
- verify:
    - `sudo systemctl status xrdp.service`
    - `sudo ss -antup|grep xrdp`