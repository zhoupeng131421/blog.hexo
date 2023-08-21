---
title: CentOS7 config xrdp
date: 2020-10-9
tags: [xrdp, centos, ubuntu]
categories: 服务配置
---


## CentOS
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

## Ubuntu
- `sudo apt install xrdp`
- `sudo adduser xrdp ssl-cert`
- `sudo systemctl restart xrdp`
- `sudo vim /etc/xrd/startwm.sh`
```shell
fi

unset DBUS_SESSION_BUS_ADDRESS
unset XDG_RUNTIME_DIR
.$HOME/.profile

if test -r /etc/profile; then
```