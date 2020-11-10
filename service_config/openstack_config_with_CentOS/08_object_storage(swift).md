---
title: OpenStack 8 object_storage(swift)
date: 2020-10-21
tags: [OpenStack, swift]
categories: 服务配置
---

# controller node
## prepare
- openstack user create --domain default --password-prompt swift
- openstack role add --project service --user swift admin
- openstack service create --name swift --description "OpenStack Object Storage" object-store
- openstack endpoint create --region RegionOne object-store public http://controller:8080/v1/AUTH_%\(project_id\)s
- openstack endpoint create --region RegionOne object-store internal http://controller:8080/v1/AUTH_%\(project_id\)s
- openstack endpoint create --region RegionOne object-store admin http://controller:8080/v1

## install & config
- yum install openstack-swift-proxy python-swiftclient python-keystoneclient python-keystonemiddleware memcached
- curl -o /etc/swift/proxy-server.conf https://opendev.org/openstack/swift/raw/branch/stable/stein/etc/proxy-server.conf-sample
- vim /etc/switf/proxy-server.conf
```shell
[DEFAULT]
...
bind_port = 8080
user = swift
swift_dir = /etc/swift

[pipeline:main]
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server

[app:proxy-server]
use = egg:swift#proxy
...
account_autocreate = True

[filter:keystoneauth]
use = egg:swift#keystoneauth
...
operator_roles = admin,user

[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_id = default
user_domain_id = default
project_name = service
username = swift
password = SWIFT_PASS
delay_auth_decision = True

[filter:cache]
use = egg:swift#memcache
...
memcache_servers = controller:11211
```

# object node
> sdb and sdc

## prepare
- yum install xfsprogs rsync
- mkfs.xfs /dev/sdb
- mkfs.xfs /dev/sdc
- mkdir -p /srv/node/sdb
- mkdir -p /srv/node/sdc
- vim /etc/fstab
```shell
/dev/sdb /srv/node/sdb xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
/dev/sdc /srv/node/sdc xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
```
- mount /srv/node/sdb
- mount /srv/node/sdc
- vim /etc/rsyncd.conf
```shell
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
# manager network ipaddr
address = MANAGEMENT_INTERFACE_IP_ADDRESS

[account]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/account.lock

[container]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/container.lock

[object]
max connections = 2
path = /srv/node/
read only = False
lock file = /var/lock/object.lock
```
- systemctl enable rsyncd.service
- systemctl start rsyncd.service

## install & config
- yum install openstack-swift-account openstack-swift-container openstack-swift-object
- curl -o /etc/swift/account-server.conf https://opendev.org/openstack/swift/raw/branch/stable/stein/etc/account-server.conf-sample
- curl -o /etc/swift/container-server.conf https://opendev.org/openstack/swift/raw/branch/stable/stein/etc/container-server.conf-sample
- curl -o /etc/swift/object-server.conf https://opendev.org/openstack/swift/raw/branch/stable/stein/etc/object-server.conf-sample
- vim /etc/swift/account-server.conf
```shell
[DEFAULT]
...
# manager network ipaddr
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6202
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True

[pipeline:main]
pipeline = healthcheck recon account-server

[filter:recon]
use = egg:swift#recon
recon_cache_path = /var/cache/swift

[filter:healthcheck]
use = egg:swift#healthcheck
```
- vim /etc/swift/container-server.conf
```shell
[DEFAULT]
...
# manager network ipaddr
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6201
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True

[pipeline:main]
pipeline = healthcheck recon container-server

[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift

[filter:healthcheck]
use = egg:swift#healthcheck
```
- vim /etc/swift/object-server.conf
```shell
[DEFAULT]
...
# manager network ipaddr
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6200
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = True

[pipeline:main]
pipeline = healthcheck recon object-server

[filter:recon]
use = egg:swift#recon
recon_cache_path = /var/cache/swift
recon_lock_path = /var/lock

[filter:healthcheck]
use = egg:swift#healthcheck
```
- chown -R swift:swift /srv/node
- mkdir -p /var/cache/swift
- chown -R root:swift /var/cache/swift
- chmod -R 775 /var/cache/swift

# Create and distribute initial rings
## controller node
### account ring
- cd /etc/swift
- create the base account.builder: `swift-ring-builder account.builder create 10 2 1`
- Add each storage node to the ring: `swift-ring-builder account.builder add --region 1 --zone 1 --ip object_address --port 6202 --device DEVICE_NAME --weight DEVICE_WEIGHT`
    - `swift-ring-builder account.builder add --region 1 --zone 1 --ip 192.168.1.45 --port 6202 --device sdb --weight 100`
    - `swift-ring-builder account.builder add --region 1 --zone 1 --ip 192.168.1.45 --port 6202 --device sdc --weight 100`
    - verify: `swift-ring-builder account.builder`
    - `swift-ring-builder account.builder reblance`
### container ring
- `swift-ring-builder container.builder create 10 2 1`
- `swift-ring-builder container.builder add --region 1 --zone 1 --ip 192.168.1.45 --port 6201 --device sdb --weight 100`
- `swift-ring-builder container.builder add --region 1 --zone 1 --ip 192.168.1.45 --port 6201 --device sdc --weight 100`
- `swift-ring-builder container.builder`
- `swift-ring-builder container.builder rebalance`
### object ring
- `swift-ring-builder object.builder create 10 2 1`
- `swift-ring-builder object.builder add --region 1 --zone 1 --ip 192.168.1.45 --port 6200 --device sdb --weight 100`
- `swift-ring-builder object.builder add --region 1 --zone 1 --ip 192.168.1.45 --port 6200 --device sdc --weight 100`
- `swift-ring-builder object.builder`
- `swift-ring-builder object.builder rebalance`

- cp account.ring.gz container.ring.gz object.ring.gz block:/etc/swift/ && run 

# Finalize installation
- controller node
    - general random HASH: `head -c 32 /dev/random | base64`
    - vim /etc/swift/swift.conf:
    ```shell
    [swift-hash]
    swift_hash_path_suffix = xxx
    swift_hash_path_prefix = xxx

    [storage-policy:0]
    name = Policy-0
    default = yes
    ```
- copy switf.conf to every object node:/etc/swift
- all node: 
    - chown -R root:swift /etc/swift
- controller node:
    - systemctl enable openstack-swift-proxy.service memcached.service
    - systemctl start openstack-swift-proxy.service memcached.service
- object node:
    - systemctl enable openstack-swift-account.service openstack-swift-account-auditor.service openstack-swift-account-reaper.service openstack-swift-account-replicator.service
    - systemctl start openstack-swift-account.service openstack-swift-account-auditor.service openstack-swift-account-reaper.service openstack-swift-account-replicator.service
    - systemctl enable openstack-swift-container.service openstack-swift-container-auditor.service openstack-swift-container-replicator.service openstack-swift-container-updater.service
    - systemctl start openstack-swift-container.service openstack-swift-container-auditor.service openstack-swift-container-replicator.service openstack-swift-container-updater.service
    - systemctl enable openstack-swift-object.service openstack-swift-object-auditor.service openstack-swift-object-replicator.service openstack-swift-object-updater.service
    - systemctl start openstack-swift-object.service openstack-swift-object-auditor.service openstack-swift-object-replicator.service openstack-swift-object-updater.service

# verify
- . admin-openrc
- swift stat
- openstack container create container1
- openstack object create container1 file(a local file name)
- openstack object list container1
- openstack object save container1 file(download to local)