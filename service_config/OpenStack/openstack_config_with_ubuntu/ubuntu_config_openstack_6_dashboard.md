---
title: OpenStack 6 dashboard
date: 2020-10-21
tags: [OpenStack, dashboard]
categories: 服务配置
---

> configurate in the controller node

- apt install openstack-dashboard
- vim `/etc/openstack-dashboard/local_settings.py`
```shell
OPENSTACK_HOST = "controller"
ALLOWED_HOSTS = ['one.example.com', 'two.example.com']      # Do not edit it under the Ubuntu.
SESSION_ENGINE = 'django.contrib.sessions.backends.file'

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

#If you chose networking option 1, disable support for layer-3 networking services:
OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_ipv6': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}

TIME_ZONE = "Asia/Shanghai"
```

- vim `WSGIApplicationGroup %{GLOBAL}`: add `WSGIApplicationGroup %{GLOBAL}`
- service apache2 reload

- verify: http://controller/horizon
    - domain: Default
    - user: admin
    - password: ADMIN_PASS