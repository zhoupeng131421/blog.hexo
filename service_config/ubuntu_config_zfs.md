---
title: ubuntu config zfs
date: 2022-05-08
tags: [zfs, ubuntu]
categories: 服务配置
---

- sudo apt install zfsutils
- create pool
    - sudo zpool create your-pool /dev/sdc /dev/sdd
    - sudo zpool create your-pool mirror /dev/sdc /dev/sdd
    - sudo zpool create your-pool raidz1 /dev/sda /dev/sdb /dev/sdc /dev/sdd
    - sudo zpool create your-pool raidz2 /dev/sdc /dev/sdd /dev/sde /dev/sdf
    - sudo zpool create your-pool mirror /dev/sdc /dev/sdd mirror /dev/sde /dev/sdf
- sudo zpool status
- sudo zpool import
    - sudo zpool import name



sudo zpool create rz1-01 raidz1 /dev/sda /dev/sdb /dev/sdc /dev/sdd