---
title: OVF export ESXi virtual machine
date: 2021-1-14
tags: [VMware, OVF]
categories: 随笔
---

> Download "VMware OVF Tool" and installed. Default installed path is `C:\Program Files\VMware\VMware OVF Tool\`.

- `cd c:\Program...\..\`
- `.\ovftool.exe vi://root:@192.168.0.150/Oracle D:`
    - ESXi host ip: `192.168.0.150`
    - virtual machine name is: `Oracle`
    - Download target path is: `D:`
```shell
Enter login information for source vi://192.168.0.150/
Username: root
Password: ************
Opening VI source: vi://root@192.168.0.150:443/Oracle
Opening OVF target: D:
Writing OVF package: D:\Oracle\Oracle.ovf
Disk progress: 9%
```
