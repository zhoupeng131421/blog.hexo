---
title: H3C交换机配置1 基本知识
date: 2019-11-1
tags: [H3C, 交换机]
categories: Learning
---

### 命令视图
```shell
  - 用户视图 - 系统视图 - 接口视图
                       - VLAN视图
                       - 用户线程视图
                       - ...
                       - 本地用户视图
```

- 用户登录后，直接进入用户视图：\<sysname>
- `system-view` 命令可以进入系统视图:[sysname]
- 系统视图下输入特定命令，进入相应功能视图，完成功能配置。
- `return` 可以从任意视图返回用户视图

### uodo
- 可以用`undo`取消某个设定：
```shell
  info-center enable
  undo info-center enable
```

### 命令行输入
- 可以使用`<tab>`快速补全命令

#### 接口类型的输入
- 使用接口类型的全称时，支持不完整的字符输入
- 使用接口类型简称时，必须输入完整的简称
- 两种方式输入的接口类型均不区分大小写
```shell
interface gigabitethernet 1/0/1:
  interface gig 1/0/1
  interface ge 1/0/1
```
- 接口类型全程、简称对应表

| interface type name | interface type abbreviation |
|---|---|
| Bridge-Aggregation | BAGG |
| GigabitEthernet| GE|
|InLoopBack| InLoop|
|LoopBack| Loop|
|M-GigabitEthernet| MGE|
|NULL|NULL|
|Ten-GigabitEthernet| XGE|
|Vlan-interface| Vlan-int|

#### 配置命令别名
- 当用户在执行别名命令时，如果别名命令中定义了参数，则参数必须输入完全，设备才会按照替换后的命令执行相关操作；否则设备将会返回命令输入不完整的提示信息，并显示出当前别名代表的命令字符串。
- 系统定义的缺省别名无法取消。
- `display ip routing-table` 别名配置为 `shiprt`: `alias shiprt display ip routing-table`
- `save` 命令保存当前配置文件。

### FIPS
> FIPS(Federal Information Processing Standards，联邦信息处理标准) 是NIST(National Institute of Standards and Technology，美国国家标准与技术研究院)颁布的针对密码算法安全的一个标准，它规定了一个安全系统中的密码模块应该满足的安全性要求。

- 不支持Telnet HTTP FTP模式登陆
- `fips mode enable`系统提供选择启动模式，默认手动启动。
- FIPS进入前，自动删除非FIPS模式配置的秘钥以及不符合FIPS标准的数字证书。非FIPS进入FIPS后，无法直接通过SSH方式登陆，需在FPIS模式下通过console口登陆，创建SSH秘钥。
