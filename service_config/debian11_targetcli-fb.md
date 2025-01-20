---
title: debian targetcli-fb
date: 2022-04-02
tags: [iSCSI, debian]
categories: 服务配置
---

- sudo apt -y install targetcli-fb
- sudo mkdir /var/lib/iscsi_disks
- sudo targetcli
```shell
# create backstore
/> cd backstores/block
/backstores/block>  create pve-iscsi /dev/zvol/rz1-01/pve-iscsi
Created block storage object pve-iscsi using /dev/zvol/rz1-01/pve-iscsi.

# create a target
/> cd /iscsi
/iscsi> create iqn.2022-06.md60dzzc.ad001.siemens.net:pve.target01
Created target iqn.2022-06.md60dzzc.ad001.siemens.net:pve.target01.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.

# set LUN
/iscsi> cd iqn.2022-06.md60dzzc.ad001.siemens.net:pve.target01/tpg1/luns
/iscsi/iqn.20...t01/tpg1/luns> create /backstores/block/pve-iscsi
Created LUN 0.

# set portals
/iscsi/iqn.20...t01/tpg1/luns> cd ../portals
/iscsi/iqn.20.../tpg1/portals> delete 0.0.0.0 3260
Deleted network portal 0.0.0.0:3260
/iscsi/iqn.20.../tpg1/portals> create 192.168.1.32 3260
Using default IP port 3260
Created network portal 192.168.1.32:3260.

# set ACL (it's the IQN of an initiator you permit to connect)
/iscsi/iqn.20...t01/tpg1/acls>  create iqn.2022-06.md60dzzc.ad001.siemens.net:pve.initiator01
Created Node ACL for iqn.2022-06.md60dzzc.ad001.siemens.net:pve.initiator01
Created mapped LUN 0.

/> ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 1]
  | | o- pve-iscsi ..................................................... [/dev/zvol/rz1-01/pve-iscsi (10.0TiB) write-thru activated]
  | |   o- alua ................................................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2022-06.md60dzzc.ad001.siemens.net:pve.target01 ............................................................... [TPGs: 1]
  |   o- tpg1 ............................................................................................... [no-gen-acls, no-auth]
  |     o- acls .......................................................................................................... [ACLs: 0]
  |     o- luns .......................................................................................................... [LUNs: 1]
  |     | o- lun0 ................................................ [block/pve-iscsi (/dev/zvol/rz1-01/pve-iscsi) (default_tg_pt_gp)]
  |     o- portals .................................................................................................... [Portals: 1]
  |       o- 192.168.1.32:3260 ................................................................................................ [OK]
  o- loopback ......................................................................................................... [Targets: 0]
  o- vhost ............................................................................................................ [Targets: 0]
  o- xen-pvscsi ....................................................................................................... [Targets: 0]

# set ACL
/iscsi/iqn.20...t01/tpg1/luns> cd ../
/iscsi/iqn.20...target01/tpg1> set attribute authentication=0
Parameter authentication is now '0'.
/iscsi/iqn.20...target01/tpg1> set attribute generate_node_acls=1
Parameter generate_node_acls is now '1'.

/iscsi/iqn.20...target01/tpg1> exit
```
- further config:
```shell
# set ACL (it's the IQN of an initiator you permit to connect)
/iscsi/iqn.20...t01/tpg1/acls>  create iqn.2022-06.md60dzzc.ad001.siemens.net:pve.initiator01
Created Node ACL for iqn.2022-06.md60dzzc.ad001.siemens.net:pve.initiator01
Created mapped LUN 0.
/iscsi/iqn.20...t01/tpg1/acls> cd iqn.2022-06.md60dzzc.ad001.siemens.net:pve.initiator01/

# set UserID and Password for authentication
/iscsi/iqn.20...w.initiator01> set auth userid=username 
Parameter userid is now 'username'.
/iscsi/iqn.20...w.initiator01> set auth password=password 
Parameter password is now 'password'.
/iscsi/iqn.20...w.initiator01> exit 
Global pref auto_save_on_exit=true
Configuration saved to /etc/rtslib-fb-target/saveconfig.json
```
- ss -napt | grep 3260
- sudo systemctl restart rtslib-fb-targetctl
- sudo systemctl status rtslib-fb-targetctl
- sudo systemctl enable rtslib-fb-targetctl

# client
- iscsiadm -m discovery -tst -p 192.168.1.32
- iscsiadm -m node -T iqn.2022-06.md60dzzc.ad001.siemens.net:pve.target01 -l

-  systemctl restart iscsid.service
- iscsiadm -m session
- iscsiadm -m node -T iqn.2022-06.md60dzzc.ad001.siemens.net:pve.target01 -p 192.168.1.32:3260 -u

```txt
sudo iscsiadm -m session
iscsiadm -m node -p 172.183.10.43 -T iqn.2024-2.cnfocpveiscsi.ad001.siemens.net:pve -u
```