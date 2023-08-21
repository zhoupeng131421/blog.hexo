---
title: ubuntu install xface
date: 2022-09-16
tags: [ubuntu, xface]
categories: 服务配置
---

# xface install
- `sudo apt update`
- `sudo apt upgrade`
- `sudo apt install xubuntu-desktop`
    - Default display manager: `gdm3`
- `reboot`
- 登陆界面输密码时，右下角选择 `Xubuntu Session`

# Issues
- 登陆界面显示用户名跟实际用户名不一致
    - `sudo vim /etc/passwd`
        - cute:x:1000:1000:`ubuntu`:/home/cute:/bin/bash
        - cute:x:1000:1000:`cute`:/home/cute:/bin/bash
- root 用户认证失败
    - `sudo passwd`
    - `sudo vim /etc/pam.d/gdm-autologin`
        - #auth required pam_succeed_if.so user != root quiet_success
    - `sudo vim /etc/pam.d/gdm-password`
        - #auth required pam_succeed_if.so user != root quiet_success
    - `sudo vim /root/.profile`
        - `mesg n || true` --> `tty -s&&mesg n || true`
    - `reboot`