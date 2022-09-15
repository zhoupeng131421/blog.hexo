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

sudo ufw route allow in on eth1 out on eth2
sudo ufw route allow in on eth0 out on eth1 to 12.34.45.67 port 80 proto tcp

sudo ufw allow proto tcp from 139.23.144.192/27 port 443 to 139.24.118.44
```

## command
- sudo ufw status numbered
- sudo ufw delete [num]

## NAT
1. `ufw default allow forward` or vim /etc/default/ufw: `DEFAULT_FORWARD_POLICY="ACCEPT"`
2. vim /etc/ufw/ssyctl.conf: `net/ipv4/ip_forward=1`
3. `ufw route allow in on enp6s16 out on enp6s18` or
vim /etc/ufw/befroe.rules:
```shell
*nat
:PREROUTING - [0:0]
:POSTROUTING - [0:0]

# enp6s16:2089 --> 192.168.15.20:3389
#-A PREROUTING -p tcp --dport exposed_port -j REDIRECT --to-port effective_port
-A PREROUTING -i enp6s16 -p tcp --dport 2089 -j DNAT --to-destination 192.168.15.20:3389
-A POSTROUTING -s 192.168.15.0/24 -o enp6s16 -j MASQUERADE

COMMIT
```

4. `ufw disable && ufw enable` or `ufw reload`



- port scan tool
    - nmap
    - zenmap (nmap GUI version)