---
title: CentOS7 config xrdp
date: 2020-10-9
tags: [xrdp, centos]
categories: 服务配置
---

- yum install epel-release
- yum install xrdp
- yum install tigervnc-server
- vim /etc/xrdp/xrdp.ini: map_bpp=32 --> 24
- selinux:
    - chcon -t bin_t /usr/sbin/xrdp
    - chcon -t bin_t /usr/sbin/xrdp-sesman
- systemd:
    - systemctl start xrdp
    - systemctl enable xrdp
- firewalld:
    - firewall-cmd  --permanent --zone=public --add-port=3389/tcp
    - firewall-cmd --reload
- verify:
    - systemctl status xrdp.service
    - ss -antup|grep xrdp