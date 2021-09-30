---
title: OpenStack HA nova config
date: 2021-3-30
tags: [OpenStack, HA]
categories: 服务配置
---

# prepare
- `mysql -uroot -p`
```shell
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'xxx';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'xxx';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'xxx';
flush privileges;
```

- create endpoint:
```shell
source admin-openrc
openstack user create --domain default --password 7Pbjqyyf nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://10.10.10.69:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://10.10.10.69:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://10.10.10.69:8774/v2.1
```

# config nova-api
- [controller01~ ]#:
```shell
openstack-config --set /etc/nova/nova.conf DEFAULT enabled_apis  osapi_compute,metadata
openstack-config --set /etc/nova/nova.conf DEFAULT my_ip  10.10.10.51
openstack-config --set /etc/nova/nova.conf DEFAULT use_neutron  true
openstack-config --set /etc/nova/nova.conf DEFAULT firewall_driver  nova.virt.firewall.NoopFirewallDriver
openstack-config --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:MQ_PASS@openstack-controller01:5672,openstack:MQ_PASS@openstack-controller02:5672,openstack:MQ_PASS@openstack-controller03:5672
openstack-config --set /etc/nova/nova.conf DEFAULT osapi_compute_listen_port 8774
openstack-config --set /etc/nova/nova.conf DEFAULT metadata_listen_port 8775
openstack-config --set /etc/nova/nova.conf DEFAULT metadata_listen '$my_ip'
openstack-config --set /etc/nova/nova.conf DEFAULT osapi_compute_listen '$my_ip'
openstack-config --set /etc/nova/nova.conf api auth_strategy  keystone
openstack-config --set /etc/nova/nova.conf api_database  connection  mysql+pymysql://nova:xxx@10.10.10.69/nova_api
openstack-config --set /etc/nova/nova.conf cache backend oslo_cache.memcache_pool
openstack-config --set /etc/nova/nova.conf cache enabled True
openstack-config --set /etc/nova/nova.conf cache memcache_servers openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set /etc/nova/nova.conf database connection  mysql+pymysql://nova:xxx@10.10.10.69/nova
openstack-config --set /etc/nova/nova.conf keystone_authtoken www_authenticate_uri  http://10.10.10.69:5000/v3
openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_url  http://10.10.10.69:5000/v3
openstack-config --set /etc/nova/nova.conf keystone_authtoken memcached_servers  openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_type  password
openstack-config --set /etc/nova/nova.conf keystone_authtoken project_domain_name  Default
openstack-config --set /etc/nova/nova.conf keystone_authtoken user_domain_name  Default
openstack-config --set /etc/nova/nova.conf keystone_authtoken project_name  service
openstack-config --set /etc/nova/nova.conf keystone_authtoken username  nova
openstack-config --set /etc/nova/nova.conf keystone_authtoken password  xxx
openstack-config --set /etc/nova/nova.conf vnc enabled  true
openstack-config --set /etc/nova/nova.conf vnc server_listen  '$my_ip'
openstack-config --set /etc/nova/nova.conf vnc server_proxyclient_address  '$my_ip'
openstack-config --set /etc/nova/nova.conf vnc novncproxy_host '$my_ip'
openstack-config --set /etc/nova/nova.conf vnc novncproxy_port  6080
openstack-config --set /etc/nova/nova.conf glance  api_servers  http://10.10.10.69:9292
openstack-config --set /etc/nova/nova.conf oslo_concurrency lock_path  /var/lib/nova/tmp
openstack-config --set /etc/nova/nova.conf placement region_name  RegionOne
openstack-config --set /etc/nova/nova.conf placement project_domain_name  Default
openstack-config --set /etc/nova/nova.conf placement project_name  service
openstack-config --set /etc/nova/nova.conf placement auth_type  password
openstack-config --set /etc/nova/nova.conf placement user_domain_name  Default
openstack-config --set /etc/nova/nova.conf placement auth_url  http://10.10.10.69:5000/v3
openstack-config --set /etc/nova/nova.conf placement username  placement
openstack-config --set /etc/nova/nova.conf placement password  PLACEMENT_PASS
```
- note:
    - 注意几个密码：MQ_PASS, PLACEMENT_PASS
    - 前端采用haproxy时，服务连接rabbitmq会出现连接超时重连的情况，可通过各服务与rabbitmq的日志查看；
    - transport_url=rabbit://openstack:Zx******@10.15.253.88:5672
    - rabbitmq本身具备集群机制，官方文档建议直接连接rabbitmq集群；但采用此方式时服务启动有时会报错，原因不明；如果没有此现象，建议连接rabbitmq直接对接集群而非通过前端haproxy的vip+端口
        - openstack-config --set /etc/nova/nova.conf DEFAULT transport_url rabbit://openstack:xxx@openstack-controller01:5672,openstack:xxx@openstack-controller02:5672,openstack:xxx@openstack-controller03:5672
- `scp -rp /etc/nova/nova.conf openstack-controller02:/etc/nova/`
- `scp -rp /etc/nova/nova.conf openstack-controller03:/etc/nova/`
- [controller02~ ]# `sed -i "s#10.10.10.51#10.10.10.52#g" /etc/nova/nova.conf`
- [controller03~ ]# `sed -i "s#10.10.10.51#10.10.10.53#g" /etc/nova/nova.conf`
- sync database:
    - `su -s /bin/sh -c "nova-manage api_db sync" nova`
    - `su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova`
    - `su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova`
    - `su -s /bin/sh -c "nova-manage db sync" nova`
    - verify:
        - `su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova`
        - `mysql -h openstack-controller01 -u nova -p -e "use nova_api;show tables;"`
        - `mysql -h openstack-controller01 -u nova -p -e "use nova;show tables;"`
        - `mysql -h openstack-controller01 -u nova -p -e "use nova_cell0;show tables;"`

- systemctl restart openstack-nova-api.service 
- systemctl restart openstack-nova-scheduler.service 
- systemctl restart openstack-nova-conductor.service 
- systemctl restart openstack-nova-novncproxy.service
- systemctl status openstack-nova-api.service 
- systemctl status openstack-nova-scheduler.service 
- systemctl status openstack-nova-conductor.service 
- systemctl status openstack-nova-novncproxy.service
- verify: 
    - `netstat -tunlp | egrep '8774|8775|8778|6080'`
    - `curl http://10.10.10.69:8774`
    - `openstack compute service list`
    - `openstack catalog list`
    - `nova-status upgrade check`

# pcs source
- `pcs resource create openstack-nova-api systemd:openstack-nova-api clone interleave=true`
- `pcs resource create openstack-nova-scheduler systemd:openstack-nova-scheduler clone interleave=true`
- `pcs resource create openstack-nova-conductor systemd:openstack-nova-conductor clone interleave=true`
- `pcs resource create openstack-nova-novncproxy systemd:openstack-nova-novncproxy clone interleave=true`
- note: openstack-nova-api，openstack-nova-conductor与openstack-nova-novncproxy 等无状态服务以active/active模式运行；openstack-nova-scheduler等服务以active/passive模式运行

# config compute nodes
- apply to every compute node.
- yum install -y openstack-nova-compute openstack-utils
- yum install -y epel-release
- yum install -y qemu
- backup conf file:
    - cp /etc/nova/nova.conf{,.bak}
    - grep -Ev '^$|#' /etc/nova/nova.conf.bak > /etc/nova/nova.conf
- get hardware info: `egrep -c '(vmx|svm)' /proc/cpuinfo`
- config nove:
```shell
openstack-config --set  /etc/nova/nova.conf DEFAULT enabled_apis  osapi_compute,metadata
openstack-config --set  /etc/nova/nova.conf DEFAULT transport_url  rabbit://openstack:xxx@10.10.10.69
openstack-config --set  /etc/nova/nova.conf DEFAULT my_ip 10.10.10.54
openstack-config --set  /etc/nova/nova.conf DEFAULT use_neutron  true
openstack-config --set  /etc/nova/nova.conf DEFAULT firewall_driver  nova.virt.firewall.NoopFirewallDriver
openstack-config --set  /etc/nova/nova.conf api auth_strategy  keystone
openstack-config --set  /etc/nova/nova.conf  keystone_authtoken www_authenticate_uri  http://10.10.10.69:5000
openstack-config --set  /etc/nova/nova.conf keystone_authtoken auth_url  http://10.10.10.69:5000
openstack-config --set  /etc/nova/nova.conf keystone_authtoken memcached_servers  openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set  /etc/nova/nova.conf keystone_authtoken auth_type  password
openstack-config --set  /etc/nova/nova.conf keystone_authtoken project_domain_name  Default
openstack-config --set  /etc/nova/nova.conf keystone_authtoken user_domain_name  Default
openstack-config --set  /etc/nova/nova.conf keystone_authtoken project_name  service
openstack-config --set  /etc/nova/nova.conf keystone_authtoken username  nova
openstack-config --set  /etc/nova/nova.conf keystone_authtoken password  xxx
openstack-config --set  /etc/nova/nova.conf libvirt virt_type  qemu
openstack-config --set  /etc/nova/nova.conf vnc enabled  true
openstack-config --set  /etc/nova/nova.conf vnc server_listen  0.0.0.0
openstack-config --set  /etc/nova/nova.conf vnc server_proxyclient_address  '$my_ip'
openstack-config --set  /etc/nova/nova.conf vnc novncproxy_base_url http://10.10.10.69:6080/vnc_auto.html
openstack-config --set  /etc/nova/nova.conf glance api_servers  http://10.10.10.69:9292
openstack-config --set  /etc/nova/nova.conf oslo_concurrency lock_path  /var/lib/nova/tmp
openstack-config --set  /etc/nova/nova.conf placement region_name  RegionOne
openstack-config --set  /etc/nova/nova.conf placement project_domain_name  Default
openstack-config --set  /etc/nova/nova.conf placement project_name  service
openstack-config --set  /etc/nova/nova.conf placement auth_type  password
openstack-config --set  /etc/nova/nova.conf placement user_domain_name  Default
openstack-config --set  /etc/nova/nova.conf placement auth_url  http://10.10.10.69:5000/v3
openstack-config --set  /etc/nova/nova.conf placement username  placement
openstack-config --set  /etc/nova/nova.conf placement password  xxx

```
- systemctl restart libvirtd.service openstack-nova-compute.service
- systemctl enable libvirtd.service openstack-nova-compute.service
- systemctl status libvirtd.service openstack-nova-compute.service

- add compute node:
    - `openstack compute service list --service nove-compute`
    - found compute node: `su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova`
    - auto found:
        - `openstack-config --set  /etc/nova/nova.conf scheduler discover_hosts_in_cells_interval 600`
        - `systemctl restart openstack-nova-api.service`    
    - verify: 
        - `openstack compute service list`
        - `openstack catalog list`
        - `openstack image list`
        - `nova-status upgrade check`