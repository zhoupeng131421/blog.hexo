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
- ssh-keygen -t rsa -f ~/.ssh/id_rsa -P ''
- ssh-copy-id -i root@ceph-mon-02
- ssh-copy-id -i root@ceph-mon-02
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
- ceph config set mon public_network xxx.xxx.xxx.xxx/xx

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

# config SSD cache
- ceph osd tree
```shell
ID  CLASS  WEIGHT    TYPE NAME            STATUS  REWEIGHT  PRI-AFF
-1         20.92267  root default
-3          8.18709      host pve-ceph01
 5    hdd   7.27739          osd.5            up   1.00000  1.00000
 0    ssd   0.90970          osd.0            up   1.00000  1.00000
-5          8.18709      host pve-ceph02
 6    hdd   7.27739          osd.6            up   1.00000  1.00000
 1    ssd   0.90970          osd.1            up   1.00000  1.00000
-7          4.54849      host pve-ceph03
 3    hdd   1.81940          osd.3            up   1.00000  1.00000
 4    hdd   1.81940          osd.4            up   1.00000  1.00000
 2    ssd   0.90970          osd.2            up   1.00000  1.00000
```

- ceph osd crush class ls
```shell
[
    "ssd",
    "hdd"
]
```

- ceph osd crush rule create-replicated ssd_rule default host ssd
- ceph osd crush rule create-replicated rule-nvme default host nvme
- ceph osd crush rule create-replicated rule-hdd default host hdd
- ceph osd crush rule list
```shell
replicated_rule
ssd_rule
rule-nvme
rule-hdd
```

- create pools:
  - `ceph osd pool create pve-ceph-rbd 64 64`
  - `ceph osd pool create cache 64 64 ssd_rule`
## config cache
  - cache 放置到 data 前: `ceph osd tier add pve-ceph-rbd cache`
  - cache mode to writeback: `ceph osd tier cache-mode cache writeback`
  - `ceph osd tier set-overlay pve-ceph-rbd cache`
  - verify: `ceph osd dump | egrep 'pve-ceph-rbd|cache'`
## 缓冲池相关参数:
1. 命中集合过滤器，默认为 Bloom 过滤器
```shell
ceph osd pool set cache hit_set_type bloom
ceph osd pool set cache hit_set_count 1
# 设置 Bloom 过滤器的误报率
ceph osd pool set cache hit_set_fpp 0.15
# 设置缓存有效期,单位：秒
ceph osd pool set cache hit_set_period 3600   # 1 hour
```

2. 设置当缓存池中的数据达到多少个字节或者多少个对象时，缓存分层代理就开始从缓存池刷新对象至后端存储池并驱逐
```shell
# 当缓存池中的数据量达到1TB时开始刷盘并驱逐
ceph osd pool set cache target_max_bytes 1099511627776

# 当缓存池中的对象个数达到100万时开始刷盘并驱逐
ceph osd pool set cache target_max_objects 10000000
```

3. 定义缓存层将对象刷至存储层或者驱逐的时间：
```shell
ceph osd pool set cache cache_min_flush_age 600
ceph osd pool set cache cache_min_evict_age 600 
```

4. 定义当缓存池中的脏对象（被修改过的对象）占比达到多少(百分比)时，缓存分层代理开始将object从缓存层刷至存储层
  - `ceph osd pool set cache cache_target_dirty_ratio 0.4`
5. 当缓存池的饱和度达到指定的值，缓存分层代理将驱逐对象以维护可用容量，此时会将未修改的（干净的）对象刷盘：
  - `ceph osd pool set cache cache_target_full_ratio 0.8`
6. 设置在处理读写操作时候，检查多少个 HitSet，检查结果将用于决定是否异步地提升对象（即把对象从冷数据升级为热数据，放入快取池）。它的取值应该在 0 和 hit_set_count 之间， 如果设置为 0 ，则所有的对象在读取或者写入后，将会立即提升对象；如果设置为 1 ，就只检查当前 HitSet ，如果此对象在当前 HitSet 里就提升它，否则就不提升。 设置为其它值时，就要挨个检查此数量的历史 HitSet ，如果此对象出现在 min_read_recency_for_promote 个 HitSet 里的任意一个，那就提升它。
```shell
ceph osd pool set cache min_read_recency_for_promote 1
ceph osd pool set cache min_write_recency_for_promote 1
```

## remove cache
1. 将缓存模式更改为转发，以便新的和修改的对象刷新至后端存储池：
  - ceph osd tier cache-mode cache forward --yes-i-really-mean-it
2. 查看缓存池以确保所有的对象都被刷新（这可能需要点时间）：
  - rados -p cache ls 
3. 如果缓存池中仍然有对象，也可以手动刷新：
  - rados -p cache cache-flush-evict-all
4. 删除覆盖层，以使客户端不再将流量引导至缓存：
  - ceph osd tier remove-overlay pve-ceph-rbd
5. 解除存储池与缓存池的绑定：
  - ceph osd tier remove pve-ceph-rbd cache

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

# command:
- remove osd:
  - ceph orch osd rm <osd_id(s)>
  - ceph orch osd rm status
- remove host:
  - ceph orch host rm xxx
  - ceph orch host ls
- remove mds:
  - ceph mds fail xxx
  - ceph mds rm xxx
- mgr:
  - ceph config-key set mgr/dashboard/server_addr <IP/hostname>
  - ceph config-key set mgr/dashboard/server_port <port>
  - ceph mgr services

- config:
  - ceph osd pool get <pool name> min_size
  - ceph osd pool get <pool name> size
  - ceph osd pool set <pool name> size x


# config dashboard
- apt install ceph-mgr-dashboard
- ceph mgr module enable dashboard
- ceph dashboard create-self-signed-cert
- ceph dashboard ac-user-create admin -i xx administrator
- ceph mgr services

# arch warning
- ceph crash ls-new
- ceph crash archive <crash-id>
- ceph crash archive-all