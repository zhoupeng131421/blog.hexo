---
title: 二层交换基本配置
date: 2019-10-31
tags: [H3C, 交换机]
categories: Learning
---

#二层交换机基本配置

## 默认配置：

|配置项|IP|CDP|10/100端口|生成树协议|console<br>password | port status|
|---|---|---|---|---|---|---|
|配置值|0.0.0.0|启动|自动协商|启动|none|启动|
| description |数据链路层工作，<br>基于MAC地址转发；<br>IP地址用于管理，<br>不用与通讯|cisio自有的发现协议<br>用于维护设备||STP协议，物理环路时，在逻辑上避免环路|||

- 交换机（二层）配置IP，目的是实现通过网络进行管理
- 交换机配置默认网关，目的实现跨网段管理

## 端口命名规则
- show running-config
- show spanning-tree
- show vlan
- show interface status
- show ip interface brief
