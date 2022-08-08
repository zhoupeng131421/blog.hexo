# server
- yum install targetd targetcli

- targetcli:
```shell
/backstores/block> create disk1 /dev/sdb

/iscsi> create iqn.2022-03.xx.xx.com:target01

/iscsi/iqn..../tpg1/luns> create /backstores/block/disk1

/iscsi/iqn..../tpg1/acls> create iqn.2022-03.xx.xx.com:iscsi01

/> saveconfig
/> exit
```

- systemctl restart targetd
- systemctl enable targetd

- netstat -lntp | grep 3260

# client
- yum install iscsi-initiator-utils
- vim /etc/iscsi/initiatorname.iscsi
    - InitiatorName=iqn.2022-03.xx.xx.com:iscsi01
- iscsiadm -m discovery -t st -p xx.xx.xx.xx:3260
    - xx.xx.xx.xx:3260,1 iqn.2022-03.xx.xx.com:target01
- iscsiadm -m node -T iqn.2022-03.xx.xx.com:target01 -l