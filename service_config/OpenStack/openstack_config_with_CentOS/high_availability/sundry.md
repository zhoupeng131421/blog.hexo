|service|web|username|password|
|---|---|---|---|
|host||root|zp@131421|
|mysql||root|zp@131421|
|mariadb clustercheck||clustercheck|123456|
|rabbitmq|http://10.10.10.51:15672|openstack|zp@131421|
|pacemaker|https://10.10.10.51:2224|hacluster|zp@131421|
|pacemaker HA|https://10.10.10.69:2224|hacluster|zp@131421|
|haproxy|http://10.10.10.69:1080|admin|admin|
|keystone||admin|w1zlcjmK|
|keystone||demo|123456|
|glance||glance|56KtIfqc|
|placement||placement|58PGTpgC|
|nova||nova|7Pbjqyyf|
|neutron||neutron|6ZKOyWXt|
|horazion|||GzDgYC2m|
|cinder|||PGZoNz7H|
||||RoPi4gRc|

|hostname|IP|basic servers|
|----|----|----|
|openstack-controller01|10.10.10.51|ChronyServer<br>DBServer<br>MessageQueue<br>Memcached<br>etcd<br>pacemaker<br>HAProxy|
|openstack-controller02|10.10.10.52||
|openstack-controller03|10.10.10.53||
|openstack-compute01|10.10.10.54||

|hostname|IP|openstack service|
|----|----|----|
|openstack-controller01|10.10.10.51|1. keystone<br>2. glance-api, glance-registry<br>3. nova-api, nova-conductor, nova-consoleauth, nova-scheduler, nova-novncproxy<br>4. neutron-api, neutron-linuxbridge-agent, neutron-dhcp-agent, neutron-metadata-agent, neutron-l3-agent<br>5. cinder-api, cinder-schedulera<br>6. dashboard|
|openstack-controller02|10.10.10.52||
|openstack-controller03|10.10.10.53||
|openstack-compute01|10.10.10.54||