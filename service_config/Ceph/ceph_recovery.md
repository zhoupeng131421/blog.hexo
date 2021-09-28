---
title: ceph recovery
date: 2021-3-11
tags: [ceph]
categories: 服务配置
---

现象：
整个ceph集群，只保留了，数据节点的磁盘(或文件夹)上的数据。其他的所有数据(key，conf等)全部丢失、或毁坏。

恢复步骤：
A 清理mon节点的相关的内容，注意使用的账号为手动新建的账号这里是cephuser
1. ceph-deploy purge mon11
2. ceph-deploy purgedata mon11
3. ceph-deploy forgetkeys
4. rm -rf ceph*
5. yum clean all

B 初始化集群并修改配置文件
1. ceph-deploy new mon11 –fsid XXXXXXXXXXXXXXXXXXXX
其中XXXX为原有集群的fsid，该id存在于osd节点的ceph_fsid中
2. vi ceph.conf修改配置文件，为原始的文件，注意去掉所有的授权cephx改为none

C安装初始化，下面2步如果出错，加—overwrite-conf
1.ceph-deploy install mon11
2.ceph-deploy mon create-initial
3.ceph-deploy admin mon11 osd11 osd22

D重新准备，激活osd节点
1.ceph-deploy osd prepare osd11:vdb osd22:vdb
2.ceph-deploy osd activate osd11:vdb osd22:vdb

E检查文件
/etc/ceph/ceph.conf。集群中所有的机器，都应该使用相同的配置文件，不一致手动改正即可。另外在ceph-deploy的执行文件夹中也有该文件，一并检查，修改。

F检查文件
/etc/ceph/ceph.client.admin.keyring文件。集群中所有的机器，都应该使用相同的keyring文件，不一致手动改正即可。另外在ceph-deploy的执行文件夹中也有该文件，一并检查，修改。

G恢复mon的相关数据
1.mkdir /tmp/mon-store ##在所有的节点机器上执行
2.ceph-objectstore-tool --data-path /vdb/ceph-0/ --op update-mon-db --mon-store-path /tmp/mon-store/ ##在osd.0的节点上执行2,3命令
3.rsync -avz /tmp/mon-store/* osd22:/tmp/mon-store
4.ceph-objectstore-tool --data-path /vdb/ceph-1/ --op update-mon-db --mon-store-path /tmp/mon-store/ ##在osd.1的节点上执行4,5命令
5.rsync -avz /tmp/mon-store/* mon11:/tmp/mon-store
6.mkdir /var/lib/ceph/mon/ceph-mon11  ##在mon11的节点上执行下面的命令
7.ceph-monstore-tool /tmp/mon-store rebuild
8.cp -ra /tmp/mon-store/* /var/lib/ceph/mon/ceph-mon11
9.touch /var/lib/ceph/mon/ceph-mon11/done
10.touch /var/lib/ceph/mon/ceph-mon11/ssytemd
11.chown cephuser:cephuser -R /var/lib/ceph/mon/