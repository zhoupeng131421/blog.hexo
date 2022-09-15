---
title: PVE install Ubuntu Cloud Template
date: 2022-08-09
tags: [PVE]
categories: 服务配置
---

1. create a vm with Serial and CloudInit
2. wget http://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
3. qm importdisk 100 focal-server-cloudimg-amd64.img local --format qcow2
4. resize disk
5. modify boot squence
6. convert to template