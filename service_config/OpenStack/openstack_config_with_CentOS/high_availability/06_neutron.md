---
title: OpenStack HA neutron config
date: 2021-9-9
tags: [OpenStack, HA]
categories: 服务配置
---

# prepare
- mysql -u root -pxxx
```shell
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'xxx';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'xxx';
flush privileges;
```

- create endpoint
```shell
source admin-openrc
openstack user create --domain default --password xxx neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network
openstack endpoint create --region RegionOne network public http://10.10.10.69:9696
openstack endpoint create --region RegionOne network internal http://10.10.10.69:9696
openstack endpoint create --region RegionOne network admin http://10.10.10.69:9696
```

# controller node neutron
- Services:
    - openstack-neutron：neutron-server的包
    - openstack-neutron-ml2：ML2 plugin的包
    - openstack-neutron-linuxbridge：linux bridge network provider相关的包
    - ebtables：防火墙相关的包
    - conntrack-tools： 该模块可以对iptables进行状态数据包检查
- 全部 controller 结点：
    - yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables conntrack-tools -y
    - 备份配置文件： /etc/neutron/neutron.conf
        - cp -a /etc/neutron/neutron.conf{,.bak}
        - grep -Ev '^$|#' /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf
## neutron server config
- openstack-config at controller01(10.10.10.51):
``` shell
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT bind_host 10.10.10.51
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT core_plugin ml2
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT service_plugins router
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT allow_overlapping_ips true
    #直接连接rabbitmq集群
    openstack-config --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:zp@131421@openstack-controller01:5672,openstack:zp@131421@openstack-controller02:5672,openstack:zp@131421@openstack-controller03:5672
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT auth_strategy  keystone
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes  true
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes  true
    #启用l3 ha功能
    #openstack-config --set  /etc/neutron/neutron.conf DEFAULT l3_ha True
    #最多在几个l3 agent上创建ha router
    #openstack-config --set  /etc/neutron/neutron.conf DEFAULT max_l3_agents_per_router 3
    #可创建ha router的最少正常运行的l3 agnet数量
    #openstack-config --set  /etc/neutron/neutron.conf DEFAULT min_l3_agents_per_router 2
    #dhcp高可用，在3个网络节点各生成1个dhcp服务器
    #openstack-config --set  /etc/neutron/neutron.conf DEFAULT dhcp_agents_per_network 3
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT l3_ha True
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT max_l3_agents_per_router 2
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT min_l3_agents_per_router 1
    openstack-config --set  /etc/neutron/neutron.conf DEFAULT dhcp_agents_per_network 2

    openstack-config --set  /etc/neutron/neutron.conf database connection  mysql+pymysql://neutron:6ZKOyWXt@10.10.10.69/neutron

    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri  http://10.10.10.69:5000
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken auth_url  http://10.10.10.69:5000
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken memcached_servers  openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken auth_type  password
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken project_domain_name  default
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken user_domain_name  default
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken project_name  service
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken username  neutron
    openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken password  6ZKOyWXt

    openstack-config --set  /etc/neutron/neutron.conf nova  auth_url http://10.10.10.69:5000
    openstack-config --set  /etc/neutron/neutron.conf nova  auth_type password
    openstack-config --set  /etc/neutron/neutron.conf nova  project_domain_name default
    openstack-config --set  /etc/neutron/neutron.conf nova  user_domain_name default
    openstack-config --set  /etc/neutron/neutron.conf nova  region_name RegionOne
    openstack-config --set  /etc/neutron/neutron.conf nova  project_name service
    openstack-config --set  /etc/neutron/neutron.conf nova  username nova
    openstack-config --set  /etc/neutron/neutron.conf nova  password 7Pbjqyyf

    openstack-config --set  /etc/neutron/neutron.conf oslo_concurrency lock_path  /var/lib/neutron/tmp
```
- scp -rp /etc/neutron/neutron.conf openstack-controller02:/etc/neutron/
- scp -rp /etc/neutron/neutron.conf openstack-controller03:/etc/neutron/
- openstack-contraller02# sed -i "s#10.10.10.51#10.10.10.52#g" /etc/neutron/neutron.conf
- openstack-contraller03# sed -i "s#10.10.10.51#10.10.10.53#g" /etc/neutron/neutron.conf
## ml2 config
- openstack-controller01:
    - cp -a /etc/neutron/plugins/ml2/ml2_conf.ini{,.bak}
    - grep -Ev '^$|#' /etc/neutron/plugins/ml2/ml2_conf.ini.bak > /etc/neutron/plugins/ml2/ml2_conf.ini
    - command:
```shell
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers  flat,vlan,vxlan
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers  linuxbridge,l2population
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers  port_security
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks  provider
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 1:1000
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset  true
```
- scp -rp /etc/neutron/plugins/ml2/ml2_conf.ini openstack-controller02:/etc/neutron/plugins/ml2/ml2_conf.ini
- scp -rp /etc/neutron/plugins/ml2/ml2_conf.ini openstack-controller03:/etc/neutron/plugins/ml2/ml2_conf.ini
## nova config
- 修改配置文件/etc/nova/nova.conf
- 在全部控制节点上配置nova服务与网络节点服务进行交互
```shell
openstack-config --set  /etc/nova/nova.conf neutron url  http://10.10.10.69:9696
openstack-config --set  /etc/nova/nova.conf neutron auth_url  http://10.10.10.69:5000
openstack-config --set  /etc/nova/nova.conf neutron auth_type  password
openstack-config --set  /etc/nova/nova.conf neutron project_domain_name  default
openstack-config --set  /etc/nova/nova.conf neutron user_domain_name  default
openstack-config --set  /etc/nova/nova.conf neutron region_name  RegionOne
openstack-config --set  /etc/nova/nova.conf neutron project_name  service
openstack-config --set  /etc/nova/nova.conf neutron username  neutron
openstack-config --set  /etc/nova/nova.conf neutron password  6ZKOyWXt
openstack-config --set  /etc/nova/nova.conf neutron service_metadata_proxy  true
openstack-config --set  /etc/nova/nova.conf neutron metadata_proxy_shared_secret  6ZKOyWXt
```
## sync nova database
- [root@controller01 ~]# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
- verify: mysql -h openstack-controller03 -u neutron -p6ZKOyWXt -e "use neutron;show tables;"
## create soft linker for ML2
- all controller node:
    - ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
## restart nova and neutron server
- all controller node:
```shell
systemctl restart openstack-nova-api.service
systemctl status openstack-nova-api.service
systemctl enable neutron-server.service
systemctl restart neutron-server.service
systemctl status neutron-server.service
```

# computer node neutron
- 由于这里部署为neutron server与neutron agent分离，所以采取这样的部署方式，常规的控制节点部署所有neutron的应用包括server和agent；
- 计算节点部署neutron agent、linuxbridge和nova配置即可；也可以单独准备网络节点进行neutron agent的部署；
## neutron service config
- yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y
- 备份配置文件/etc/nova/nova.conf
    - cp -a /etc/neutron/neutron.conf{,.bak}
    - grep -Ev '^$|#' /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf
- neutron config:
```shell
openstack-config --set  /etc/neutron/neutron.conf DEFAULT bind_host 10.10.10.54
openstack-config --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:zp@131421@openstack-controller01:5672,openstack:zp@131421@openstack-controller02:5672,openstack:zp@131421@openstack-controller03:5672
openstack-config --set  /etc/neutron/neutron.conf DEFAULT auth_strategy keystone 
#配置RPC的超时时间，默认为60s,可能导致超时异常.设置为180s
openstack-config --set  /etc/neutron/neutron.conf DEFAULT rpc_response_timeout 180

openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://10.10.10.69:5000
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken auth_url http://10.10.10.69:5000
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken memcached_servers openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken auth_type password
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken project_domain_name default
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken user_domain_name default
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken project_name service
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken username neutron
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken password 6ZKOyWXt

openstack-config --set  /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
```
- scp -rp /etc/neutron/neutron.conf openstack-compute02:/etc/neutron/
- [computer02 #] sed -i "s#10.10.10.54#10.10.10.55#g" /etc/neutron/neutron.conf
## nova config
- all compute nodes
```shell
openstack-config --set  /etc/nova/nova.conf neutron url http://10.10.10.69:9696
openstack-config --set  /etc/nova/nova.conf neutron auth_url http://10.10.10.69:5000
openstack-config --set  /etc/nova/nova.conf neutron auth_type password
openstack-config --set  /etc/nova/nova.conf neutron project_domain_name default
openstack-config --set  /etc/nova/nova.conf neutron user_domain_name default
openstack-config --set  /etc/nova/nova.conf neutron region_name RegionOne
openstack-config --set  /etc/nova/nova.conf neutron project_name service
openstack-config --set  /etc/nova/nova.conf neutron username neutron
openstack-config --set  /etc/nova/nova.conf neutron password 6ZKOyWXt
```
## plugin and agent config
### ML2
- all compute nodes
- cp -a /etc/neutron/plugins/ml2/ml2_conf.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/plugins/ml2/ml2_conf.ini.bak > /etc/neutron/plugins/ml2/ml2_conf.ini
```shell
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers  flat,vlan,vxlan
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers  linuxbridge,l2population
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers  port_security
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks  provider
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_vxlan vni_ranges 1:1000
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset  true
```
- scp -rp /etc/neutron/plugins/ml2/ml2_conf.ini openstack-compute02:/etc/neutron/plugins/ml2/ml2_conf.ini
### Linux bridge agent config
- all compute nodes
- cp -a /etc/neutron/plugins/ml2/linuxbridge_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak >/etc/neutron/plugins/ml2/linuxbridge_agent.ini
```shell
#环境无法提供四张网卡；建议生产环境上将每种网络分开配置
#provider网络对应规划的ens192
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings  provider:ens192

openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan true

#tunnel租户网络（vxlan）vtep端点，这里对应规划的ens192地址，根据节点做相应修改
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip 10.10.10.54

openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan l2_population true
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group  true
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver  neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
- scp -rp /etc/neutron/plugins/ml2/linuxbridge_agent.ini  openstack-compute02:/etc/neutron/plugins/ml2/
- [compute02 #] sed -i "s#10.10.10.54#10.10.10.55#g" /etc/neutron/plugins/ml2/linuxbridge_agent.ini 
### L3 agent config
- all compute nodes
- cp -a /etc/neutron/l3_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/l3_agent.ini.bak > /etc/neutron/l3_agent.ini
- openstack-config --set /etc/neutron/l3_agent.ini DEFAULT interface_driver linuxbridge
### DHCP agent config
- all compute nodes
- cp -a /etc/neutron/dhcp_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/dhcp_agent.ini.bak > /etc/neutron/dhcp_agent.ini
```shell
openstack-config --set  /etc/neutron/dhcp_agent.ini DEFAULT interface_driver linuxbridge
openstack-config --set  /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
openstack-config --set  /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata true
```
### Metadata agent config
- 元数据代理提供配置信息，例如实例的凭据
- metadata_proxy_shared_secret 的密码与控制节点上/etc/nova/nova.conf文件中密码一致；
- cp -a /etc/neutron/metadata_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/metadata_agent.ini.bak > /etc/neutron/metadata_agent.ini
```shell
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_host 10.10.10.69
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret 6ZKOyWXt
openstack-config --set /etc/neutron/metadata_agent.ini cache memcache_servers openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211
```
- scp -rp /etc/neutron/metadata_agent.ini openstack-compute02:/etc/neutron/
### other config
- all compute nodes
```shell
echo 'net.ipv4.ip_nonlocal_bind = 1' >>/etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-iptables=1' >>/etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables=1'  >>/etc/sysctl.conf

#启用网络桥接器支持，需要加载 br_netfilter 内核模块；否则会提示没有目录
modprobe br_netfilter
sysctl -p
```
- systemctl restart openstack-nova-compute.service
```shell
systemctl enable neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
systemctl restart neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
systemctl status neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent neutron-l3-agent
```
# verify
- [root@controller01]# openstack extension list --network
- [root@controller01]# openstack network agent list
# PCS resource
- 只需要添加neutron-server，其他的neutron-agent服务：neutron-linuxbridge-agent，neutron-l3-agent，neutron-dhcp-agent与neutron-metadata-agent 不需要添加了；因为部署在了计算节点上
- any controller node:
```shell
#pcs resource create neutron-linuxbridge-agent systemd:neutron-linuxbridge-agent clone interleave=true
#pcs resource create neutron-l3-agent systemd:neutron-l3-agent clone interleave=true
#pcs resource create neutron-dhcp-agent systemd:neutron-dhcp-agent clone interleave=true
#pcs resource create neutron-metadata-agent systemd:neutron-metadata-agent clone interleave=true
pcs resource create neutron-server systemd:neutron-server clone interleave=true
```
- verify: pcs resource