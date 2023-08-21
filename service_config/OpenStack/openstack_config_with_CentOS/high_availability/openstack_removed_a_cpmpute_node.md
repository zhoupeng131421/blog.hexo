# remove nova service
- nova service-list | grep compute:
```shell
+--------------------------------------+----------------+------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| Id                                   | Binary         | Host                   | Zone     | Status  | State | Updated_at                 | Disabled Reason | Forced down |
+--------------------------------------+----------------+------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| cba0b7ff-9b9f-430f-a1ca-accf25c559e9 | nova-compute   | openstack-compute02    | nova     | enabled | up    | 2021-10-18T03:46:07.000000 | -               | False       |
| d22ee5c3-dfcb-49df-9071-ae5b207d0f4a | nova-compute   | openstack-compute01    | nova     | enabled | up    | 2021-10-18T03:46:07.000000 | -               | False       |
| be30c74a-27b2-4856-9ebc-08f1334b0dc9 | nova-compute   | openstack-compute03    | nova     | enabled | up    | 2021-10-18T03:46:12.000000 | -               | False       |
+--------------------------------------+----------------+------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
```

- nova service-disable cba0b7ff-9b9f-430f-a1ca-accf25c559e9
- nova service-delete cba0b7ff-9b9f-430f-a1ca-accf25c559e9
- nova service-list | grep compute
```shell
| cba0b7ff-9b9f-430f-a1ca-accf25c559e9 | nova-compute   | openstack-compute02    | nova     | disabled | up    | 2021-10-18T03:49:08.000000 | -               | False       |
| d22ee5c3-dfcb-49df-9071-ae5b207d0f4a | nova-compute   | openstack-compute01    | nova     | enabled  | up    | 2021-10-18T03:49:07.000000 | -               | False       |
| be30c74a-27b2-4856-9ebc-08f1334b0dc9 | nova-compute   | openstack-compute03    | nova     | enabled  | up    | 2021-10-18T03:49:12.000000 | -               | False       |
```

- mysql -uroot -p
```shell
use nova;
delete from nova.services where host="openstack-compute02";
delete from compute_nodes where hypervisor_hostname="openstack-compute02";
select host from nova.services;
select hypervisor_hostname from compute_nodes;
```
- verify:
    - openstack host list
    - nova service-list

# remove neutron service
- [~ xx@xx_compute02] systemctl stop neutron-linuxbridge-agent.service neutron-metadata-agent.service neutron-dhcp-agent.service neutron-l3-agent.service
- [~ xx@xx_compute02] systemctl status neutron-linuxbridge-agent.service neutron-metadata-agent.service neutron-dhcp-agent.service neutron-l3-agent.service
- [~ xx@xx_compute02] systemctl disable neutron-linuxbridge-agent.service neutron-metadata-agent.service neutron-dhcp-agent.service neutron-l3-agent.service
- [~ xx@xx_controller] openstack network agent list
- [~ xx@xx_controller] openstack network agent delete xxx
- [~ xx@xx_controller] openstack network agent list
- verify: mysql -uroot -p
```shell
select * from neutron.agents where host='<HOST NAME>'\G;
select * from neutron.agents;
```

