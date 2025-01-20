---
title: iSCSI target with tgtadm
date: 2020-10-08
tags: [iSCSI]
categories: 服务配置
---

# iSCSI target config
- apt install tgt
- tgtadm --lld iscsi --mode target --op new --tid 1 --targetname iqn.2014-0728.com:longtang
```shell
#--lld iscsi 固定参数驱动iscsi
#--mode  target 模式target
#--op    new    操作new新创建
#--tid   1      target id号
#--targetname ...target名称
#查看Target记录</span>
```
- tgtadm --lld iscsi --mode target --op show
    - auto create LUN with id 0
- tgtadm --lld iscsi --mode target --op bind --tid 1 -I ALL
    - allow all initiator to connection
- tgtadm --lld iscsi --mode logicalunit --op new --tid 1 --lun 1 -b /dev/vg_xxx/lv_xxx
    - create new LUN for target according the tid
- tgtadm --lld iscsi --mode target --op show
- ufw allow iscsi-target

- create account: `tgtadm --lld iscsi --op new --mode account --user gitlab --password xxx`
- bind account to a target accouding tid: `tgtadm --lld iscsi --op bind --mode account --tid 1 --user gitlab`
- tgtadm --lld iscsi --mode target --op show
- 对 target 访问权限解绑: `tgtadm --lld iscsi --mode target --op unbind --tid 1 -I ALL`
- tgtadm --lld iscsi --mode target --op bind --tid 1 -I gitlab
- tgtadm --lld iscsi --mode target --op show

# iSCSI initiator config
- apt install open-iscsi
- 发现target: `iscsiadm -m discovery -t sendtargets -p 192.168.1.xx`
- 显示发现的tartget node: `iscsiadm -m node`
- 连接 target: `iscsiadm -m node -T iqn.2020-10.xxx -p 192.168.1.xx:3260 --login`
- 查看 SCSI status: `cat /proc/scsi/scsi`
- `lsblk`
- 格式化: `mkfs.ext4 /dev/sdb`
- mount: `mount -t ext4 /dev/sdb /mnt`
- umount: `umount /mnt`
- 断开 target: `iscsiadm -m node -p 192.168.1.xx --logout`
- 解除挂载： `iscsiadm -m node –T iqn.1997-05.com.test:raid -p 192.168.1.x:3260 –u`
- 设置开机自动挂载 target: `iscsiadm -m node –T iqn.2020-10.com.xxx -p 192.168.1.xxx:3260 --op update -n node.startup -v automatic`
- 设置开机自动挂载 disk:
    - vim /etc/fstab:
    - `/dev/sdb /mnt ext4 defaults,_netdev 0 0`

## 登陆需要认证的 iSCSI node
- vim /etc/iscis/iscsid.conf
```shell
node.session.auth.authmethod = CHAP
node.session.auth.username = gitlab
node.session.auth.password = xxx
```
- `iscsiadm -m node -T iqn.2020-10.com.siemens.fz:gitlabServerBackup -p 139.24.118.51:3260 -o update --name=node.session.auth.authmethod --value=CHAP`
    - --value根据配置的`authmethod`决定



- iscsiadm -m node
    - 列出所有发现的 target
- iscsiadm -m node -o delete all
    - 删除所有target
- iscsiadm -m node -T iqn.xxx:x -o delete
    - 删除某条target
- iscsiadm -m node -p 10.10.10.x -T iqn.xx.xx.xx:xx -u
    - 登出单条target
    - iscsiadm -m node -p 172.183.10.43 -T iqn.2024-2.cnfocpveiscsi.ad001.siemens.net:pve -u

