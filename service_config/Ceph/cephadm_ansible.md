---
title: ceph deploy with ansible
date: 2021-3-9
tags: [ceph]
categories: 服务配置
---

> reference: https://codepre.com/install-ceph-15-octopus-storage-cluster-on-ubuntu-20-04.html


# prepare
- install apps: net-tools vim python3

# ansible config
- yum install epel-release-7
- yum install ansible

- generate ssh key: ssh-keygen
- distribute ssh key:
    - ssh-copy-id -i root@ceph-mon-01
    - ssh-copy-id -i root@ceph-mon-02
    - ssh-copy-id -i root@ceph-mon-03
- vim hosts
```shell
# [cephpartners]
ceph-mon-01
ceph-mon-02
ceph-mon-03
```
- verify
	- ansible -m command -a "yum install -y net-tools" host

- vim prepare_ceph_nodes_centos.yml
```yaml
--- 
- name: prepare ceph nodes
  gather_facts: no
  hosts: all
  tasks:
    - name: Set time zone
      timezone:
        name: Asia/Shanghai
    - name: Update System
      yum:
        name: "*"
        state: latest
        update_cache: yes
    - name: Install common packages
      yum:
        name: [net-tools, vim, git, wget, curl, chrony]
        state: present
        update_cache: yes
    - name: Enable Chrony
      shell: |
        systemctl enable chronyd
        systemctl start chronyd
    - name: Disable Firewall
      shell: |
        systemctl stop firewalld
        systemctl disable firewalld
    - name: Disable SeLinux
      lineinfile:
        dest: /etc/selinux/config
        regexp: '^SELINUX='
        line: 'SELINUX=disabled'
    - name: Install Docker
      shell: |
        yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
        yum install -y yum-utils
        yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
        yum install -y docker-ce docker-ce-cli containerd.io
        systemctl enable docker
        systemctl start docker

- name: Try Install python3
  hosts: all
  gather_facts: no
  tasks:
    - name: Get python3 status
      stat:
        path: /usr/bin/python3
      register: python3_status
    - name: Try to get python3 version
      command: "python3 --version"
      register: python3_version
      when: python3_status.stat.exists == true
    - name: Install python3 if python3 not existed
      shell: |
        yum -y groupinstall "Development tools"
        yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel
        wget https://www.python.org/ftp/python/3.7.9/Python-3.7.9.tar.xz
        mkdir /usr/local/python3
        tar -xvJf  Python-3.7.9.tar.xz
        cd Python-3.7.9
        ./configure --prefix=/usr/local/python3
        make && make install
        ln -s /usr/local/python3/bin/python3 /usr/bin/python3
        ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3
      when: python3_status.stat.exists == false
    - name: Print python3 version
      debug:
        msg: "{{ python3_version.stdout }}"
      when: python3_version.changed == true
```
- ansible-playbook -i hosts prepare_ceph_nodes_centos.yml


# cephadm install
- ssh ceph-mon-01
- curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
- chmod +x cephadm
- ./cephadm add-repo --release octopus
- ./cephadm install
- install ceph-common:
    - cephadm install ceph-common
    - ceph -v
    - ceph status
- deploy the firest ceph node:
    - cephadm bootstrap --mon-ip ceph-mon-01
    - verify: https://ceph-mon-01:8443
    - ceph mgr services

# ceph config
## hosts manager
### add hosts
- ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon-02
- ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-mon-03
- ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-osd-01
- Tell Ceph that the new node is part of the cluster:
    - ceph orch host add ceph-mon-02
    - ceph orch host add ceph-mon-03
    - ceph orch host add ceph-osd-01
### removing hosts
- If the node that want you to remove is running OSDs, make sure you remove the OSDs from the node.
- ceph orch host rm xxx
- ceph orch host rm ceph-mon-02
### other command
- ceph orch host ls
- add `label` to a host:
    - ceph orch host label add ceph-mon-01 mon
    - ceph orch host label add ceph-mon-02 mon
    - ceph orch host label add ceph-mon-03 mon
    - ceph orch host label add ceph-osd-01 osd

## adding additional mons
- ceph orch apply mon *<host1,host2,host3,...>*
- ceph orch apply mon --unmanaged #否则每个新host都会成为mon节点
- ceph orch apply mon 3 #指定集群中mon数量
- ceph orch apply mon ceph-mon-02
- ceph orch apply mon ceph-mon-03
- Tell cephadm to deploy monitors based on the label by running this command:
    - ceph orch apply mon label:mon

## adding storage
- ceph orch device ls
- ceph orch apply osd --all-available-devices
  - if some disk indicate no available, run command: `ceph orch device zap host-name /dev/sdx --force` to rease a device
- ceph orch daemon add osd ceph-osd-01:/dev/sdb
- ceph orch daemon add osd ceph-osd-01:/dev/sdc
- ceph orch daemon add osd ceph-osd-01:/dev/sdd
- ceph orch daemon add osd ceph-osd-01:/dev/sde
- ceph -s

## cephFS
### create cephFS
- ceph osd pool create cephfs_data \<pg_num\>
- ceph osd pool create cephfs_metadata  \<pg_num\>
- ceph fs new my_fs cephfs_metadata cephfs_data 
- ceph fs ls
- ceph fs volume create xxx:
  - 相当于三个命令的集合：
  - ceph osd pool create xxx xxx
  - ceph osd pool create xxx xxx
  - ceph fs new xxx xxx xxx
- other commands:
  - ceph osd pool ls
  - ceph osd pool get/set fs_name pg_num
  - ceph osd pool get/set fs_name pgs_num
> 通常在创建pool之前，需要覆盖默认的pg_num，官方推荐：若少于5个OSD， 设置pg_num为128。5-10个OSD，设置pg_num为512。10-50个OSD，设置pg_num为4096。超过50个OSD，可以参考pgcalc计算

### fuse mount
- yum install ceph-fuse ceph
- ssh root@ceph-mon-01 "ceph-authtool -p /etc/ceph/ceph.client.admin.keyring" > admin.key
- chmod 600 admin.key
- mount -t ceph node1:6789:/ /mnt -o name=admin,secretfile=admin.key
- verify:
  - df -hT
- auto mount: /etf/fstab: id=admin,conf=/etc/ceph/ceph.conf  /mnt fuse.ceph defaults 0 0

## config MDS
- ceph orch apply mds *<fs-name>* --placement="*<num-daemons>* [*<host1>* ...]"
- ceph orch apply mds my_fs  --placement="ceph-mon-01 ceph-mon-02 ceph-mon-03"
- ceph mds stat

## config RGWs
- radosgw-admin realm create --rgw-realm=my_realm --default
- radosgw-admin zonegroup create --rgw-zonegroup=my_zonegroup  --master --default
- radosgw-admin zone create --rgw-zonegroup=my_zonegroup --rgw-zone=my_zone --master --default
- radosgw-admin period update --rgw-realm=my_realm --commit
- ceph orch apply rgw my_realm my_zone --placement="3 ceph-mon-01 ceph-mon-02 ceph-mon-03"

## config NFS
- ceph osd pool create my_nfs_pool \<pg_num\>
- ceph orch apply nfs my_nfs my_nfs_pool nfs-ns
- ceph osd pool application enable my_nfs_pool rbd
  - 这里我们使用了rbd（块设备），pool 只能对一种类型进行 enable，另外两种类型是cephfs（文件系统），rgw（对象存储）
- ceph osd pool application enable my_nfs_pool cephfs


# trouble shooting
## ceph-mon-01 mon down
- dashboard 中找到mon service active 的node
- ssh ceph-mon-02
- apt install cephadm
- cephadm shell -- ceph orch apply mon 3
- cephadm shell -- ceph orch apply mon ceph-mon-01


## no active mgr
> https://docs.ceph.com/en/latest/cephadm/troubleshooting/#manually-deploying-a-mgr-daemon

- Disable the cephadm scheduler, in order to prevent cephadm from removing the new MGR: `ceph config-key set mgr/cephadm/pause true`
- Then get or create the auth entry for the new MGR: `ceph auth get-or-create mgr.ceph-mon-01.smfvfd mon "profile mgr" osd "allow *" mds "allow *"`
  - my hostname is ceph-mon-01
```shell
[mgr.ceph-mon-01.smfvfd]
        key = AQC7H1JglNruHxAAc9b9S1oygbja0BzKs/i8Jg==
```
- Get the ceph.conf: `ceph config generate-minimal-conf`
```shell
# minimal ceph.conf for b26a161c-86f1-11eb-a43b-c5f85b4bdf41
[global]
        fsid = b26a161c-86f1-11eb-a43b-c5f85b4bdf41
        mon_host = [v2:10.10.10.21:3300/0,v1:10.10.10.21:6789/0] [v2:10.10.10.22:3300/0,v1:10.10.10.22:6789/0] [v2:10.10.10.23:3300/0,v1:10.10.10.23:6789/0]
```
- Get the container image: `ceph config get "mgr.ceph-mon-01.smfvfd" container_image`
```shell
docker.io/ceph/ceph:v15
```
- Create a file `config-json.json` which contains the information neccessary to deploy the daemon:
  - the config and keyring data from previce steps
```shell
{
  "config": "# minimal ceph.conf for b26a161c-86f1-11eb-a43b-c5f85b4bdf41\n[global]\n\tfsid = b26a161c-86f1-11eb-a43b-c5f85b4bdf41f\n\tmon_host = [v2:10.10.10.21:3300/0,v1:10.10.10.21:6789/0] [v2:10.10.10.22:3300/0,v1:10.10.10.22:6789/0] [v2:10.10.10.23:3300/0,v1:10.10.10.23:6789/0]\n",
  "keyring": "[mgr.ceph-mon-01.smfvfd]\n\tkey = AQC7H1JglNruHxAAc9b9S1oygbja0BzKs/i8Jg==\n"
}
```
- Deploy the daemon: `cephadm --image docker.io/ceph/ceph:v15 deploy --fsid b26a161c-86f1-11eb-a43b-c5f85b4bdf41 --name mgr.ceph-mon-01.smfvfd --config-json config-json.json`