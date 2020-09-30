---
title: H3C交换机配置2 登陆
date: 2019-11-3
tags: [H3C, 交换机]
categories: Learning
---

# 登陆
- 认证方式：

| 认证方式 | 描述 |
| --- | --- |
| none | 不需用户名密码，FIPS模式下不允许 |
| password | 密码正确才能登陆, FIPS模式下不支持 |
| scheme | 登陆时需要用户名和密码 |

## 一、CLI 登陆
> CLI登陆模式包括console Telnet SSH

### 1. console 口本地登陆
- none 模式配置：

```shell
system-view
line aux first-number #用户线视图： 高优先级，只对该用户生效
line class aux  # 用户线类视图： 下次登陆后生效

authentication-mode none
user-role role-name # 缺省登陆角色为 network-admin
```

- password 模式配置：

```shell
...
authentication-mode password
set authentication passoword {hash|simple}password
user-role role-name
```

- scheme 模式配置：

```shell
...
authentication-mode scheme
```

- 公共属性：

```shell
speed speed-value
parity {even|mark|none|odd|space}
activation-key character
lock-key key-string
...
```

### 2. Telnet 登陆
> 认证方式为password，设备为配置缺省密码，因此需要通过console开启Telnet，然后对认证方式、用户角色、公共属性进行配置，才能保证Telnet正常登陆。

- 配置步骤：

```shell
system-view
telnet server enable
line vty first-number   # 进入一个或多个VTY用户线视图
line class vty    # 进入用户线类视图
authentication-mode none/password/scheme
user-role role-name

aaa session-limit telnet max-sessions # 配置最大同时在线用户数
telnet server dscp dscp-value # 配置Telnet服务器发送DSCP优先级
telnet server ipv6 dscp dscp-value

shell     # 启动终端服务
idle-timeout  # config time out value
```

- 做为 Telnet 客户端登陆其他设备

```shell
telnet clent source {interface interface-type interface-number|ip-address}    #指定 Telnet 为客户端
telnet remote-host[service-port][source{interface interface-type interface-number|ip ipaddress}|dscp dscp-value]*
```

### 3. SSH 登陆
- SSH serve config:

```shell
system-view
public local create {dsa|ecdsa[secp192r1|...]|rsa} [name key-name] # no FIPS
public-key local create {dsa|ecdsa[...]|rsa} [name key-name] # FIPS
ssh server enable

line vty first-number
line class vty
authentication-mode scheme
protocol inbound {all|ssh|telnet} # no FIPS
protocol inbound ssh  # FIPS
```

- SSH 客户端登陆其他设备：`ssh2 servrer` or `ssh2 ipv6 server`

### 4. other config
```shell
display users
display users all
display line [num1 | {aux | vty}] [summary]
display telnet client
free line {num1 | {aux | vty} num2}
lock
lock reauthentication
send {all | num1 | {aux | vty} num2}
```

## 二、WEB登陆
> HTTP and HTTPS

### HTTP
- config:
```shell
web captcha verification-code
ip http enable
ip http port port-number  # 80
web idle-timeout minutes
webui log enable
local-user user-name [class manage]
authentication-attribute user-role user-role
service-type http
```

### HTTPS
- config:
```shell
ip https enable
ip https port port-number # 443
loca-user user-name [class manage]
authorization-attribute user-role user-role
service-type https
```
- display web config
```shell
display web users
display web menu [ chinese ]
display ip http
display ip https
free web users {all | user-id user-id | user-name user-name}
```

### 示例：
#### HTTP
- 条件：PC与设备通过IP相连且路由可达，PC和设备Vlan-interface1的IP分别是`192.168.101.99/24` `192.168.100.99/24 `
- graph：
```mermaid
graph LR;
  PC:192.168.101.99/24 --- |NetWork| Device:192.168.100.99/24
```
- config:
```shell
[Sysname] local-user admin
[Sysname-luser-manage-admin] service-type http
[Sysname-luser-manage-admin] authorization-attribute user-role network-admin
[Sysname-luser-manage-admin] password simple admin
[Sysname-luser-manage-admin] quit
[Sysname] ip http enable
```

#### HTTPS
- condition 1：Device作为HTTPS服务器，并为Device申请证书
- condition 2：HTTPS客户端Host申请证书，以便Device验证其身份
- condition 3：Windows Server 作为 CA，安装SCEP插件
- graph:
```mermaid
graph LR;
  Host:10.1.1.2/24 --- |10.1.1.1/24|Device
  --- |10.1.2.1/24|CA:10.1.2.2/24
```
- config:
```shell
system-view
pki entity en
common-name http-server1
fqdn ssl.security.com
quit

pki domain 1
ca identifier new-ca
certificate request url https://10.1.2.2/certsrv/mscep/mscep.dll
certificate request from ra
certificate request entiry en
public-key rsa general name hostkey length 1024
quit

public-key local cerate rsa
pki retrieve-certificate domain 1 ca
pki request-certificate domain 1

ssl server-policy myssl
pki-domain 1
client-verify enable
quit

pki certificate attribute-group mygroup1
attribute 1 issuer-name dn ctn new-ca
quit

pki certificate access-control-policy myacp
rule 1 permit mygroup1
quit

ip https ssl-server-policy myssl
ip https certificate access-control-policy myacp
ip https enable

local-user usera
password simple 123
service-type https
authorization-attribute user-role network-admin
```
- Host 上输入网址`http://10.1.2.2.certsrv`申请证书
- Host 输入网址`https://10.1.1.1`，选择new-ca，验证结果
