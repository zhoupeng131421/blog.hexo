CentOS
	controller : openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables conntrack-tools
	computer   : openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables
	network    : 
	
	controller : openstack-neutron openstack-neutron-linuxbridge
	computer   : openstack-neutron-linuxbridge
	netwrok    : openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ...



Ubuntu
	controller : neutron-server neutron-linuxbridge-agent neutron-plugin-ml2 neutron-l3-agent neutron-dhcp-agent  neutron-metadata-agent
	computer   : neutron-linuxbridge-agent
-->
	controller : neutron-server neutron-linuxbridge-agent
	computer   : neutron-linuxbridge-agent
	network    : neutron-server neutron-linuxbridge-agent neutron-plugin-ml2 neutron-l3-agent neutron-dhcp-agent  neutron-metadata-agent
	
all nodes:
yum install yum-utils -y 
yum config-manager --set-enabled ha powertools
yum install centos-release-openstack-victoria -y
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

yum clean all
yum makecache

systemctl disable firewalld
sed -i 's/enforcing/disabled/' /etc/selinux/config

reboot

modprobe br_netfilter
echo 'net.ipv4.ip_forward = 1' >>/etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-iptables=1' >>/etc/sysctl.conf
echo 'net.bridge.bridge-nf-call-ip6tables=1'  >>/etc/sysctl.conf
echo 'net.ipv4.ip_nonlocal_bind = 1' >>/etc/sysctl.conf
sysctl -p



yum install python3-openstackclient -y
mkdir -p /opt/tools
yum install crudini -y
wget -P /opt/tools https://cbs.centos.org/kojifiles/packages/openstack-utils/2017.1/1.el7/noarch/openstack-utils-2017.1-1.el7.noarch.rpm
rpm -ivh /opt/tools/openstack-utils-2017.1-1.el7.noarch.rpm

yum install -y net-tools vim git wget curl chrony
vim /etc/chrony.conf: server fz-controller01 iburst
systemctl enable chronyd
systemctl restart chronyd
chronyc sources


yum install curl python3 podman -y
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
./cephadm add-repo --release octopus
./cephadm install

controller: yum install python3-rbd -y
compute: yum install ceph-common -y




controller nodes:

yum install -y pacemaker pcs corosync fence-agents resource-agents haproxy
yum install -y mariadb-server python2-PyMySQL rabbitmq-server memcached python3-memcached etcd

yum install mariadb mariadb-server python2-PyMySQL -y
yum install mariadb-server-galera mariadb-galera-common galera xinetd rsync -y
systemctl restart mariadb.service
systemctl enable mariadb.service


Ceph:
all:
yum install curl python3 podman -y
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
./cephadm add-repo --release octopus
./cephadm install

controller: yum install python3-rbd -y
compute: yum install ceph-common -y

	
services:
systemctl status mariadb.service xinetd rabbitmq-server etcd pcsd haproxy httpd.service openstack-glance-api.service openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler neutron-server.service memcached.service openstack-cinder-api.service openstack-cinder-scheduler.service openstack-cinder-volume.service target.service


su -s /bin/sh -c "keystone-manage db_sync" keystone
su -s /bin/sh -c "glance-manage db_sync" glance
su -s /bin/sh -c "placement-manage db sync" placement
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
su -s /bin/sh -c "cinder-manage db sync" cinder




openstack-config --set /etc/nova/nova.conf DEFAULT block_device_allocate_retries 180











openstack image create --file ~/cirros-0.5.1-x86_64-disk.img --disk-format qcow2 --container-format bare --public cirros-qcow2
openstack image create --disk-format qcow2 --container-format bare --public --file ~/cirros-0.5.1-x86_64-disk.img cirros-qcow2


openstack image create --container-format bare --disk-format iso --public win10-ltsc.iso  --file cn_windows_10_multiple_editions_version_1703_updated_march_2017_x64_dvd_10194190.iso 

openstack image create --container-format bare --disk-format iso --public virtio-win-0.1.208.iso  --file virtio-win-0.1.208.iso 
openstack flavor create --ram 4096 --disk 5 --ephemeral 50 --vcpus 2 --public install-iso
openstack flavor create --ram 4096 --disk 1 --ephemeral 10 --vcpus 2 --public install-xp-iso
openstack flavor create --ram 4096 --disk 16 --ephemeral 0 --vcpus 2 --public cpu2-ram4-disk16
openstack flavor create --ram 4096 --disk 200 --ephemeral 0 --vcpus 4 --public cpu4-ram4-disk200
openstack flavor create --ram 4096 --disk 500 --ephemeral 0 --vcpus 4 --public cpu4-ram4-disk500
openstack flavor create --ram 8192 --disk 200 --ephemeral 0 --vcpus 4 --public cpu4-ram8-disk200

nova boot --image d897d4e1-cb9e-4fe0-831a-2f4179b6d544 --flavor 39dd58a8-6b47-457e-be2f-9909c2665a68 --block-device id=0c663412-8ff2-4d9d-8edb-d1e39ffd69a7,source=image,dest=volume,bus=fdc,type=floppy,size=1 --nic net-id=07b84500-c373-4358-a63c-ef841bc0a04a xp-installer

nova boot --image d897d4e1-cb9e-4fe0-831a-2f4179b6d544 --flavor 39dd58a8-6b47-457e-be2f-9909c2665a68 --block-device id=b69d42c7-e84a-434a-9b7d-6b841742f10a,source=image,dest=volume,bus=fdc,type=floppy,size=1 --nic net-id=07b84500-c373-4358-a63c-ef841bc0a04a xp-installer


nova boot --image bef40f1f-480d-49a2-ae8a-24f4d188a8c0 --flavor 03332fa6-754a-4060-98f2-355a9912e195 --block-device id=0c663412-8ff2-4d9d-8edb-d1e39ffd69a7,source=image,dest=volume,bus=ide,type=cdrom,size=1 --nic net-id=bd87659e-2491-4047-a8f0-27ba110a04e0 win7-installer


nova boot --image 5fe0fc92-43aa-4b50-87ee-c7b5548bd641 --flavor 03332fa6-754a-4060-98f2-355a9912e195 --block-device id=0c663412-8ff2-4d9d-8edb-d1e39ffd69a7,source=image,dest=volume,bus=ide,type=cdrom,size=1 --nic net-id=bd87659e-2491-4047-a8f0-27ba110a04e0 win10-installer

31247c34cb8f4ed3960a90ceef52f0b3 | normal worker | default
