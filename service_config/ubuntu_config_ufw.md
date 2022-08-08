---
title: ubuntu config ufw
date: 2019-10-15
tags: ubuntu ufw
categories: 随笔
---

# ubuntu ufw config

## 默认规则
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
- 默认禁止一切外部访问，如果SSH登录，应先设置`allow ssh`，不然就登录不了了。
`sudo ufw allow ssh` or `sudo ufw sllow 22`
```
sudo ufw enable   # open firewall
sudo ufw status   # cat firewall status
```

## 其他常用规则
```
sudo ufw allow http   # port 80
sudo ufw allow https  # port 443
sudo ufw allow 5000:6000/tcp  # tcp port from 5000 to 6000
sudo ufw allow 5000:6000/udp  # udp port from 5000 to 6000

sudo ufw deny http    # port 80
sudo ufw deny from 10.10.10.10  # deny any request from 10.10.10.10
sudo ufw allow in on eno1 from 139.24.118.0/25   # 允许 139.24.118.0/25 的机器通过eno1 访问

sudo ufw delete deny 80 # delete a rule
sudo ufw reset        # reset config(delete all rules)
sudo ufw disable      # close firewall
```

```
sudo ufw default deny incoming   # 默认关入
sudo ufw default allow outgoing  # 默认开出
sudo ufw allow in on enp94s0
sudo ufw allow proto tcp from 139.24.118.0/25 to 139.24.118.44 port 22      # ssh
sudo ufw allow proto tcp from 139.24.118.0/25 to 139.24.118.44 port 443     # https
sudo ufw allow proto tcp from 139.24.118.0/25 to 139.24.118.44 port 139     # samba
sudo ufw allow proto tcp from 139.24.118.0/25 to 139.24.118.44 port 445     # samba

sudo ufw allow proto tcp from 139.23.144.192/27 port 443 to 139.24.118.44
```

## command
- sudo ufw status numbered
- sudo ufw delete [num]