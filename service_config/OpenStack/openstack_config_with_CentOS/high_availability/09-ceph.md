---
title: OpenStack HA ceph-cluster config
date: 2021-9-14
tags: [OpenStack, HA]
categories: 服务配置
---

# Intro
- 网络：
    - public: 10.10.10.x/24
    - culster network: 172.31.10.253.x/24
    - ceph-mon-01: 10.10.10.21 172.31.10.21
    - ceph-mon-02: 10.10.10.22 172.31.10.22
    - ceph-mon-03: 10.10.10.23 172.31.10.23
# prepare
- essential config:
```shell
#（1）关闭防火墙：
systemctl stop firewalld
systemctl disable firewalld

#（2）关闭selinux：
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0

#（5）设置文件连接数最大值
echo "ulimit -SHn 102400" >> /etc/rc.local
cat >> /etc/security/limits.conf << EOF
* soft nofile 65535
* hard nofile 65535
EOF

#（6）内核参数优化
echo 'net.ipv4.ip_forward = 1' >>/etc/sysctl.conf
echo 'kernel.pid_max = 4194303' >>/etc/sysctl.conf
#内存不足时低于此值，使用交换空间
echo "vm.swappiness = 0" >>/etc/sysctl.conf 
sysctl -p

#（7）同步网络时间和修改时区；已经添加不需要配置
安装chrony时间同步 同步cephnode01节点
yum install chrony -y
vim /etc/chrony.conf 
server cephnode01 iburst
---
systemctl restart chronyd.service 
systemctl enable chronyd.service 
chronyc sources

#(8)read_ahead,通过数据预读并且记载到随机访问内存方式提高磁盘读操作
echo "8192" > /sys/block/sda/queue/read_ahead_kb

#(9) I/O Scheduler，SSD要用noop(电梯式调度程序)，SATA/SAS使用deadline(截止时间调度程序)
#https://blog.csdn.net/shipeng1022/article/details/78604910
echo "deadline" >/sys/block/sda/queue/scheduler
echo "deadline" >/sys/block/sdb/queue/scheduler
#echo "noop" >/sys/block/sd[x]/queue/scheduler

# 安装基础软件
yum install net-tools wget vim bash-completion lrzsz unzip zip -y
yum install python3 podman -y
```

# cephadm deploy
- [root@ceph-mon-01 ~]# :
- curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
- chmod +x cephadm
- ./cephadm add-repo --release octopus
- ./cephadm install
- install ceph-common:
    - cephadm install ceph-common
    - ceph -v
    - ceph status
- deploy the firest ceph node:
    - cephadm bootstrap --mon-ip 10.10.10.21
    - verify: https://ceph-mon-01:8443
    - ceph mgr services
- ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon-02
- ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon-03
- Tell Ceph that the new node is part of the cluster:
    - ceph orch host add ceph-mon-02
    - ceph orch host add ceph-mon-03
- ceph orch host label add ceph-mon-01 mon
- ceph orch host label add ceph-mon-02 mon
- ceph orch host label add ceph-mon-03 mon
- ceph orch apply mon label:mon

- ceph orch device ls
- ceph orch apply osd --all-available-devices
  - if some disk indicate no available, run command: `ceph orch device zap host-name /dev/sdx --force` to rease a device
- ceph orch daemon add osd ceph-osd-01:/dev/sdb
- ceph orch daemon add osd ceph-osd-01:/dev/sdc
- ceph orch daemon add osd ceph-osd-01:/dev/sdd
- ceph orch daemon add osd ceph-osd-01:/dev/sde
- ceph -s

- ceph orch apply mds fs-cluster --placement=3

# RGW
- create a realm:
- radosgw-admin realm create --rgw-realm=rgw-org --default
- radosgw-admin zonegroup create --rgw-zonegroup=rgwgroup --master --default
- radosgw-admin zone create --rgw-zonegroup=rgwgroup --rgw-zone=zone-dc1 --master --default
- ceph orch apply rgw rgw-org zone-dc1 --placement="2 ceph-mon-02 ceph-mon-03"

- radosgw-admin user create --uid=admin --display-name=admin --system
```shell
{
    "user_id": "admin",
    "display_name": "admin",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "subusers": [],
    "keys": [
        {
            "user": "admin",
            "access_key": "Q3TTSEQQY2MMFOSMFKIC",
            "secret_key": "jXKGev0zxgddlYT23fEgQT1jtQSGZ1JUhLJL6VuU"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "system": "true",
    "default_placement": "",
    "default_storage_class": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}
```
- ceph dashboard set-rgw-api-access-key -i access-key(save the access_key to this file)
- ceph dashboard set-rgw-api-secret-key -i secret-key
- further config:
    - ceph dashboard set-rgw-api-ssl-verify False
    - ceph dashboard set-rgw-api-scheme http
    - ceph dashboard set-rgw-api-host 10.10.10.23(ceph-mon-03)
    - ceph dashboard set-rgw-api-port 80
    - ceph dashboard set-rgw-api-user-id admin
- restart server: ceph orch restart rgw

