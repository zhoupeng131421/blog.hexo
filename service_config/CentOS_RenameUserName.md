- pkill -u old
- pkill -9 -u old

- usermod -l new old
- mv /home/old /home/new
- usermod -d /home/new -m new
- groupmod -n new old

- /etc/samba/smb.conf and /etc/samba/passwd.bd:
```shell
z003xmae --> liyunying

```


- changeusername.sh
```shell
#!/bin/bash

echo $#
if [ $# -ne 2 ];then
    echo "please input old and new user name"
    exit
fi

old=$1
new=$2


echo "old: $old, new: $new"
#sed -i "s#$old#$new#g" passwd
#sed -i "s#$old#$new#g" /etc/passwd
#sed -i "s#$old#$new#g" /etc/group
sed -i "s#$old#$new#g" /etc/shadow
#mv /home1/$old /home1/$new
#sed -i "s#$old#$new#g" /etc/samba/smb.conf
#sed -i "s#$old#$new#g" /etc/samba/smbpasswd

```
