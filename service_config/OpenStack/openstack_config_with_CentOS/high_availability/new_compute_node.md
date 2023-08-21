## prepare
- yum config-manager --set-enabled powertools
- yum install centos-release-openstack-victoria
- yum install yum-utils -y 
- rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
- yum clean all
- yum makecache
- yum upgrade -y
- yum install python3-openstackclient -y
- openstack-utils:
```shell
mkdir -p /opt/tools
yum install crudini -y
wget -P /opt/tools https://cbs.centos.org/kojifiles/packages/openstack-utils/2017.1/1.el7/noarch/openstack-utils-2017.1-1.el7.noarch.rpm
rpm -ivh /opt/tools/openstack-utils-2017.1-1.el7.noarch.rpm
```

- yum install -y net-tools vim git wget curl chrony
- chrony:
    - vim /etc/chrony.conf: server fz-controller01 iburst
    - systemctl enable chronyd.service
    - systemctl start chronyd.service
    - verify: chronyc sources
- seLinux:
    - vim /etc/selinux/config: 'SELINUX=disabled'
    - setenforce 0
    - getenforce
- firewall:
    - systemctl stop firewalld
    - systemctl disable firewalld
-  optimize kernel parameter
```shell
modprobe br_netfilter
echo 'net.ipv4.ip_forward = 1' >>/etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-iptables=1' >>/etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables=1'  >>/etc/sysctl.conf
echo 'net.ipv4.ip_nonlocal_bind = 1' >>/etc/sysctl.conf
sysctl -p
```

## nova
- yum install -y openstack-nova-compute
- backup conf file:
    - cp /etc/nova/nova.conf{,.bak}
    - grep -Ev '^$|#' /etc/nova/nova.conf.bak > /etc/nova/nova.conf
- get hardware info: `egrep -c '(vmx|svm)' /proc/cpuinfo`
- config nove:
```shell
openstack-config --set  /etc/nova/nova.conf DEFAULT enabled_apis  osapi_compute,metadata
openstack-config --set  /etc/nova/nova.conf DEFAULT transport_url  rabbit://openstack:123456@fz-controller01:5672,openstack:123456@fz-controller02:5672,openstack:123456@fz-controller03:5672
openstack-config --set  /etc/nova/nova.conf DEFAULT my_ip 10.10.10.57
openstack-config --set  /etc/nova/nova.conf DEFAULT use_neutron  trues
openstack-config --set  /etc/nova/nova.conf DEFAULT firewall_driver  nova.virt.firewall.NoopFirewallDriver
openstack-config --set  /etc/nova/nova.conf api auth_strategy  keystone
openstack-config --set /etc/nova/nova.conf  keystone_authtoken www_authenticate_url  http://10.10.10.69:5000
openstack-config --set  /etc/nova/nova.conf keystone_authtoken auth_url  http://10.10.10.69:5000
openstack-config --set  /etc/nova/nova.conf keystone_authtoken memcached_servers  fz-controller01:11211,fz-controller02:11211,fz-controller03:11211
openstack-config --set  /etc/nova/nova.conf keystone_authtoken auth_type  password
openstack-config --set  /etc/nova/nova.conf keystone_authtoken project_domain_name  Default
openstack-config --set  /etc/nova/nova.conf keystone_authtoken user_domain_name  Default
openstack-config --set  /etc/nova/nova.conf keystone_authtoken project_name  service
openstack-config --set  /etc/nova/nova.conf keystone_authtoken username  nova
openstack-config --set  /etc/nova/nova.conf keystone_authtoken password  56KtIfqc
openstack-config --set /etc/nova/nova.conf libvirt virt_type  kvm
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
openstack-config --set  /etc/nova/nova.conf placement password  56KtIfqc

```
- systemctl restart libvirtd.service openstack-nova-compute.service
- systemctl enable libvirtd.service openstack-nova-compute.service
- systemctl status libvirtd.service openstack-nova-compute.service
### add compute node:
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

## neutron
- yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y
- 备份配置文件/etc/nova/nova.conf
    - cp -a /etc/neutron/neutron.conf{,.bak}
    - grep -Ev '^$|#' /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf
- neutron config:
```shell
openstack-config --set  /etc/neutron/neutron.conf DEFAULT bind_host 10.10.10.57
openstack-config --set /etc/neutron/neutron.conf DEFAULT transport_url rabbit://openstack:123456@fz-controller01:5672,openstack:123456@fz-controller02:5672,openstack:123456@fz-controller03:5672
openstack-config --set  /etc/neutron/neutron.conf DEFAULT auth_strategy keystone 
#配置RPC的超时时间，默认为60s,可能导致超时异常.设置为180s
openstack-config --set  /etc/neutron/neutron.conf DEFAULT rpc_response_timeout 180

openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken www_authenticate_uri http://10.10.10.69:5000
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken auth_url http://10.10.10.69:5000
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken memcached_servers fz-controller01:11211,fz-controller02:11211,fz-controller03:11211
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken auth_type password
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken project_domain_name default
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken user_domain_name default
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken project_name service
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken username neutron
openstack-config --set  /etc/neutron/neutron.conf keystone_authtoken password 56KtIfqc

openstack-config --set  /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
```
- scp -rp /etc/neutron/neutron.conf openstack-compute02:/etc/neutron/
- [computer02 #] sed -i "s#10.10.10.54#10.10.10.55#g" /etc/neutron/neutron.conf
### nova config
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
openstack-config --set  /etc/nova/nova.conf neutron password 56KtIfqc
```
### plugin and agent config
#### ML2
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
#### Linux bridge agent config
- all compute nodes
- cp -a /etc/neutron/plugins/ml2/linuxbridge_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak >/etc/neutron/plugins/ml2/linuxbridge_agent.ini
```shell
#环境无法提供四张网卡；建议生产环境上将每种网络分开配置
#provider网络对应规划的ens192
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings  provider:eno1

openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan true

#tunnel租户网络（vxlan）vtep端点，这里对应规划的ens192地址，根据节点做相应修改
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan local_ip 10.10.10.57

openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan l2_population true
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group  true
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver  neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
- scp -rp /etc/neutron/plugins/ml2/linuxbridge_agent.ini  openstack-compute02:/etc/neutron/plugins/ml2/
- [compute02 #] sed -i "s#10.10.10.54#10.10.10.55#g" /etc/neutron/plugins/ml2/linuxbridge_agent.ini 
#### L3 agent config
- all compute nodes
- cp -a /etc/neutron/l3_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/l3_agent.ini.bak > /etc/neutron/l3_agent.ini
- openstack-config --set /etc/neutron/l3_agent.ini DEFAULT interface_driver linuxbridge
#### DHCP agent config
- all compute nodes
- cp -a /etc/neutron/dhcp_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/dhcp_agent.ini.bak > /etc/neutron/dhcp_agent.ini
```shell
openstack-config --set  /etc/neutron/dhcp_agent.ini DEFAULT interface_driver linuxbridge
openstack-config --set  /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
openstack-config --set  /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata true
```
#### Metadata agent config
- 元数据代理提供配置信息，例如实例的凭据
- metadata_proxy_shared_secret 的密码与控制节点上/etc/nova/nova.conf文件中密码一致；
- cp -a /etc/neutron/metadata_agent.ini{,.bak}
- grep -Ev '^$|#' /etc/neutron/metadata_agent.ini.bak > /etc/neutron/metadata_agent.ini
```shell
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_host 10.10.10.69
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret 56KtIfqc
openstack-config --set /etc/neutron/metadata_agent.ini cache memcache_servers fz-controller01:11211,fz-controller02:11211,fz-controller03:11211
```
- scp -rp /etc/neutron/metadata_agent.ini openstack-compute02:/etc/neutron/
#### other config
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
### verify
- [root@controller01]# openstack extension list --network
- [root@controller01]# openstack network agent list

## Cinder
- yum install openstack-cinder targetcli python3-keystone -y
- cp -a /etc/cinder/cinder.conf{,.bak}
- grep -Ev '#|^$' /etc/cinder/cinder.conf.bak>/etc/cinder/cinder.conf

```shell
openstack-config --set /etc/cinder/cinder.conf DEFAULT transport_url rabbit://openstack:123456@fz-controller01:5672,openstack:123456@fz-controller02:5672,openstack:123456@fz-controller03:5672
openstack-config --set /etc/cinder/cinder.conf  DEFAULT auth_strategy keystone
openstack-config --set /etc/cinder/cinder.conf  DEFAULT my_ip 10.10.10.57
openstack-config --set /etc/cinder/cinder.conf  DEFAULT glance_api_servers http://10.10.10.69:9292
openstack-config --set /etc/cinder/cinder.conf  DEFAULT enabled_backends ceph
openstack-config --set /etc/cinder/cinder.conf  database connection mysql+pymysql://cinder:56KtIfqc@10.10.10.69/cinder
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken www_authenticate_uri http://10.10.10.69:5000
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken auth_url http://10.10.10.69:5000
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken memcached_servers  fz-controller01:11211,fz-controller02:11211,fz-controller03:11211
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken auth_type password
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken project_domain_name default
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken user_domain_name default
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken project_name service
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken username cinder
openstack-config --set /etc/cinder/cinder.conf  keystone_authtoken password 56KtIfqc
openstack-config --set /etc/cinder/cinder.conf  oslo_concurrency lock_path /var/lib/cinder/tmp
```

- services: 
```shell
systemctl restart openstack-cinder-volume.service target.service
systemctl enable openstack-cinder-volume.service target.service
systemctl status openstack-cinder-volume.service target.service
```

- verify: openstack volume service list

## integrate with ceph
yum install curl python3 podman -y
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
./cephadm add-repo --release octopus
./cephadm install
cephadm install ceph-common
- ~[root@ceph-mon #]:
```shell
scp -rp /etc/ceph/ceph.client.admin.keyring compute-xg4:/etc/ceph/
scp -rp /etc/ceph/ceph.conf compute-xg4:/etc/ceph/ceph.conf
```
