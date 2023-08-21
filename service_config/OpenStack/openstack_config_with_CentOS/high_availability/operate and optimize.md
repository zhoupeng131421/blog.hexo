## normal check command
- pcs resource

## operate
- openstack security group list
a1848678-ba40-45d0-ad1e-0fe543690417
#许可ICMP协议(ping命令)
openstack security group rule create --proto icmp a1848678-ba40-45d0-ad1e-0fe543690417
#允许SSH访问(22端口)
openstack security group rule create --proto tcp --dst-port 22 a1848678-ba40-45d0-ad1e-0fe543690417
#查看安全组规则
openstack security group rule list a1848678-ba40-45d0-ad1e-0fe543690417

## compute node:
- openstack-config --set /etc/nova/nova.conf libvirt cpu_mode host-model
- openstack-config --set /etc/nova/nova.conf DEFAULT block_device_allocate_retries 180
- openstack-config --set /etc/nova/nova.conf DEFAULT ram_allocation_ratio 1.5

## controller node:
- 项目配额管理
    - openstack quota show <project ID>
    - openstack quota set <project ID> --instances 100
    - openstack quota set <project ID> --cores 100

|Quota name|Description|
|---|---|
|cores|Number of instance cores (VCPUs) allowed per project.|
|instances|Number of instances allowed per project.|
|key_pairs|Number of key pairs allowed per user.|
|gigabytes|volume buffer limit|
|metadata_items|Number of metadata items allowed per instance.|
|ram|Megabytes of instance ram allowed per project.|
|server_groups|Number of server groups per project.|
|server_group_members|Number of servers per server group.|
|fixed_ips|Number of fixed IP addresses allowed per project. This number must be equal to or greater than the number of allowed instances.|
|floating_ips|Number of floating IP addresses allowed per project.|
|networks|Number of networks allowed per project (no longer used).|
|security_groups|Number of security groups per project.|
|security_group_rules|Number of security group rules per project.|
|injected_files|Number of injected files allowed per project.|
|injected_file_content_bytes|Number of content bytes allowed per injected file.|
|injected_file_path_bytes|Length of injected file path.|

- mariadb:
```shell
[mysqld]
query_cache_type=1                              #查询缓存  (0 = off、1 = on、2 = demand)
net_read_timeout=3600                           #连接繁忙阶段（query）起作用
net_write_timeout=3600                          #连接繁忙阶段（query）起作用
key_buffer_size = 32M                           #设置索引块缓存大小
max_allowed_packet = 256M                       #通信缓冲大小
```

