---
title: ubuntu config ufw
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

sudo ufw delete deny 80 # delete a rule
sudo ufw reset        # reset config(delete all rules)
sudo ufw disable      # close firewall
```
