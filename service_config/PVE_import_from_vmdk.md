---
title: PVE import vmdk
date: 2019-10-15
tags: [vmdk, PVE]
categories: 服务配置
---

- ref: http://www.itca.cc/ProxmoxVE%E8%99%9A%E6%8B%9F%E5%8C%96/42.html

- 创建WIN10虚拟机
- copy ESXi dump出来的vmdk文件到PVE主机上
- vmdk --> qcow2
```shell
qm importdisk <vmid> <source> <storage> [OPTIONS]
qm importdisk 101 vm01-disk001.vmdk local-lvm -format qcow2
# 101: VM ID; local-lvm: storage ID; -format qcow2: image format, default is RAW.
```

- 登陆管理页面，加载磁盘
    - 打开刚才创建的虚拟机的硬件，会看到有个未使用的磁盘，双击打开，点确认即可。
    - 对应编辑一下引导顺序。
- UEFI引导迁移过来会需要更多设置才可以用。
    - 首先需要要在web配置页面中，在“选项”栏中把BIOS的值改成“OVMF(UEFI)”，再从“硬件”栏给该虚拟机加上一个“EFI磁盘”，该磁盘的作用跟电脑主板上的NVRAM差不多，就是用来存储EFI的配置信息，例如启动项列表。如果没有这个磁盘，每次配置好启动项之后，只要虚拟机一关，配置信息就会消失。
    - 然后在虚拟机启动的时候按下“ESC”键进入所谓的“BIOS”配置界面，
        - 依次选择“Boot Maintenance Manager”->"Boot Options"->"Add Boot Option"，接着会出来若干个包含了EFI分区的硬盘（一般是1个），回车键选中该硬盘，依次选择目录"<EFI>"->"redhat"->"grub.efi"，这时候会出来一个填写启动项信息的界面，我在"Input the description"中填写了“centos6.7”，然后选中"Commit Changes and Exit"。这个时候直接返回了“Boot Options”界面，选中菜单"Change Boot Order"进行启动项顺序的调整，把之前新添加的"<centos6.5>"调到最上面即可。然后选择"Commit Changes and Exit"返回刚才的界面，接着一直按“ESC”出去到最外面的界面，选择"Continue"就会成功出现centos的启动菜单了。
- 分离磁盘，再双击该为使用磁盘，选择virto总线，调整启动顺序为新的virto总线的硬盘。



