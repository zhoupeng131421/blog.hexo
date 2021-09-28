---
title: OpenStack HA horazion config
date: 2021-9-14
tags: [OpenStack, HA]
categories: 服务配置
---

> OpenStack仪表板Dashboard服务的项目名称是Horizon，它所需的唯一服务是身份服务keystone，开发语言是python的web框架Django。
> 仪表盘使得通过OpenStack API与OpenStack计算云控制器进行基于web的交互成为可能。
> Horizon 允许自定义仪表板的商标；并提供了一套内核类和可重复使用的模板及工具。

# install
- all controller node
- yum install openstack-dashboard memcached python3-memcached -y

# config
## local_settings
- cp -a /etc/openstack-dashboard/local_settings{,.bak}
- grep -Ev '^$|#' /etc/openstack-dashboard/local_settings.bak >/etc/openstack-dashboard/local_settings
- vim /etc/openstack-dashboard/local_settings
```shell
#指定在网络服务器中配置仪表板的访问位置;默认值: "/"
WEBROOT = '/dashboard/'
#配置仪表盘在controller节点上使用OpenStack服务
OPENSTACK_HOST = "10.10.10.69"

#允许主机访问仪表板,接受所有主机,不安全不应在生产中使用
ALLOWED_HOSTS = ['*', 'localhost']
#ALLOWED_HOSTS = ['one.example.com', 'two.example.com']

#配置memcached会话存储服务
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'openstack-controller01:11211,openstack-controller02:11211,openstack-controller03:11211',
    }
}

#启用身份API版本3
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

#启用对域的支持
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

#配置API版本
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

#配置Default为通过仪表板创建的用户的默认域
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

#配置user为通过仪表板创建的用户的默认角色
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

#如果选择网络选项1，请禁用对第3层网络服务的支持,如果选择网络选项2,则可以打开
OPENSTACK_NEUTRON_NETWORK = {
    #自动分配的网络
    'enable_auto_allocated_network': False,
    #Neutron分布式虚拟路由器（DVR）
    'enable_distributed_router': False,
    #FIP拓扑检查
    'enable_fip_topology_check': False,
    #高可用路由器模式
    'enable_ha_router': True,
    #下面三个已过时,不用过多了解,官方文档配置中是关闭的
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    #ipv6网络
    'enable_ipv6': True,
    #Neutron配额功能
    'enable_quotas': True,
    #rbac政策
    'enable_rbac_policy': True,
    #路由器的菜单和浮动IP功能,Neutron部署中有三层功能的支持;可以打开
    'enable_router': True,
    #默认的DNS名称服务器
    'default_dns_nameservers': [],
    #网络支持的提供者类型,在创建网络时，该列表中的网络类型可供选择
    'supported_provider_types': ['*'],
    #使用与提供网络ID范围,仅涉及到VLAN，GRE，和VXLAN网络类型
    'segmentation_id_range': {},
    #使用与提供网络类型
    'extra_provider_types': {},
    #支持的vnic类型,用于与端口绑定扩展
    'supported_vnic_types': ['*'],
    #物理网络
    'physical_networks': [],
}

#配置时区为亚洲上海
TIME_ZONE = "Asia/Shanghai"
....
```
- scp -rp /etc/openstack-dashboard/local_settings  openstack-controller02:/etc/openstack-dashboard/
- scp -rp /etc/openstack-dashboard/local_settings  openstack-controller03:/etc/openstack-dashboard/
## openstack-dashboard config
- all controller nodes
- cp /etc/httpd/conf.d/openstack-dashboard.conf{,.bak}
- ln -s /etc/openstack-dashboard /usr/share/openstack-dashboard/openstack_dashboard/conf
    - 建立策略文件（policy.json）的软链接，否则登录到dashboard将出现权限错误和显示混乱
- sed -i '3a WSGIApplicationGroup\ %{GLOBAL}' /etc/httpd/conf.d/openstack-dashboard.conf
    - 赋权，在第3行后新增 WSGIApplicationGroup %{GLOBAL}
- scp -rp /etc/httpd/conf.d/openstack-dashboard.conf  openstack-controller02:/etc/httpd/conf.d/
- scp -rp /etc/httpd/conf.d/openstack-dashboard.conf  openstack-controller03:/etc/httpd/conf.d/
```shell
systemctl restart httpd.service memcached.service
systemctl enable httpd.service memcached.service
systemctl status httpd.service memcached.service
```
# verify
- http://10.10.10.69/dashboard
    - domain: default
    - user: admin
    - pass: xxx