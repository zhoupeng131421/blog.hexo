---
title: OpenStack 7 dashboard
date: 2020-10-21
tags: [OpenStack, dashboard]
categories: 服务配置
---

## install & config
- yum install openstack-dashboard
- vim /etc/openstack-dashboard/local_settings
```shell
OPENSTACK_HOST = "controller"
# ALLOWED_HOSTS can also be [‘*’] to accept all hosts
ALLOWED_HOSTS = ['one.example.com', 'two.example.com']

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

# if not user Layer-3 agent
OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}

TIME_ZONE = "Asia/Shanghai"
```
- vim /etc/httpd/conf.d/openstack-dashboard.conf: `WSGIApplicationGroup %{GLOBAL}`
- systemctl restart httpd.service memcached.service

## verify
http://controller/dashboard