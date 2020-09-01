---
title: LVM简介及常用命令
date: 2020-9-1
tags: [LVM]
categories: 随笔
---

# LVM description
LVM是Logical Volume Manager逻辑卷管理,包括分配磁盘,以及对逻辑卷进行striping/mirroring/resizing操作.将一个或多个硬盘的分区在逻辑上集合，相当于一个大硬盘来使用，当硬盘的空间不够使用的时候，可以继续将其它的硬盘的分区加入其中，这样可以实现磁盘空间的动态管理，是建立在硬盘和分区之上、文件系统之下的一个逻辑层，相对于普通的磁盘分区有很大的灵活性:

- 使系统管理员可以更方便的为应用与用户分配存储空间
- 在LVM管理下的存储卷可以按需要随时改变大小与移除
-  LVM也允许按用户组对存储卷进行管理，允许管理员用更直观的名称(如"sales'、'development')代替物理磁盘名(如'sda'、'sdb')来标识存储卷

# Principle
## Term
- The physical media 物理存储介质：LVM存储介质，可以是硬盘分区、整个硬盘、raid阵列或SAN硬盘。设备必须初始化为LVM物理卷，才能与LVM结合使用。
- physicalvolume/PV 物理卷: 物理的磁盘分区,是lvm的基本存储逻辑块.
- Volume group/VG 卷组: 由一个或者多个物理卷组成,可以在卷组上面创建一个或多个逻辑卷.
- Logicalvolume/LV 逻辑卷: 就是从VG中划分的逻辑分区,在逻辑卷上可以创建文件系统(比如/home,/usr等)
- physical extent/PE: 每一个物理卷被划分为称为PE(PhysicalExtents)的基本单元，具有唯一编号的PE是可以被LVM寻址的最小单元。PE的大小是可配置的，默认为4MB。
- Logical extent/LE: 逻辑卷也被划分为被称为LE(LogicalExtents) 的可被寻址的基本单位。在同一个卷组中，LE的大小和PE是相同的，并且一一对应。
- ![](/assets/lvm/lvm_structure.png)
## Note
- 多个磁盘/分区/raid-->多个物理卷PV-->合成卷组VG-->从VG划分出逻辑卷LV-->格式化LV，挂载使用
- LVM是软件的卷管理方式，RAID是磁盘管理的方法。对于重要的数据，用RAID保护物理硬盘不会因为故障而中断业务，再用LVM来实现对卷的良性管理，更好的利用硬盘资源。

# USE
## command
|function|PV command|VG command|LV command|
|-|-|-|-|
|Scan|pvscan|vgscan|lvscan|
|Create|pvcreate|vgcreate|lvcreate|
|Display|pvdisplay|vgdisplay|lvdisplay|
|Remove|pvremove|vgremove|lvremove|
|Extend||vgextend|lvextend|
|Reduce||vgreduce|lvreduce|
- 查看命令有scan、display和s（pvs、vgs、lvs）。s是简单查看对应卷信息，display是详细查看对应卷信息。而scan是扫描所有的相关的对应卷。

- `pvcreate /dev/sdx`
- `vgcreate [option] [param]`
    - option: 
        - -l：卷组上允许创建的最大逻辑卷数；
        - -p：卷组中允许添加的最大物理卷数；
        - -s：卷组上的物理卷的PE大小。
    - param:
        - 卷组名：要创建的卷组名称；
        - 物理卷列表：要加入到卷组中的物理卷列表。
    - `vgcreate vg0 /dev/sdb1 /dev/sdb2  `
- `lvcreate`
    - -L 指定具体大小
    - -l 指定大小占卷组比例
    - -n lv名字
    ```shell
    #创建一个指定大小的lv，并指定名字为lv_2
    lvcreate -L 2G -n lv_2 vg_1
    #创建一个占全部卷组大小的lv，并指定名字为lv_3（注意前提是vg并没有创建有lv）
    lvcreate -l 100%VG -n lv_3 vg_1
    #创建一个空闲空间80%大小的lv，并指定名字为lv_4(常用)
    lvcreate -l 80%Free -n lv_4 vg_1
    ```