---
title: OpenStack issue
date: 2020-10-21
tags: [OpenStack]
categories: 服务配置
---

### openstack volume downloading
- Openstack虚拟机磁盘创建超时
- 1. 增大尝试次数：
    - compute node: vim /etc/nova/nova.conf: `block_dievce_allocate_retries = 180`
- 2. Image-Volume cache
    - block node: vim /etc/cinder/cinder.conf:
    ``` shell
    cinder_internal_tenant_project_id = PROJECT_ID
    cinder_internal_tenant_user_id = USER_ID
    image_volume_cache_enabled = True
    image_volume_cache_max_size_gb = SIZE_GB
    image_volume_cache_max_count = MAX_COUNT
    ```

### Ubuntu cloud image default account
- 实例时：configure 中添加 script：
```shell
#!/bin/sh
passwd ubuntu<<EOF
ubuntu
ubuntu
EOF
```