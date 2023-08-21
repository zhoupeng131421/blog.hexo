- sudo apt install sssd sssd-ad sssd-ldap sssd-krb5 libnss-winbind  libpam-sss ldap-utils libpam-ldapd libnss-ldapd sssd-tools sssd libpam-sss adcli oddjob libpam-sss adcli sssd-tools sssd samba krb5-config krb5-user winbind libpam-winbind libnss-winbind

- sudo apt install krb5-kdc krb5-admin-server
    - debian加入 windows server的 ADS域，需要安装 windows server 并开启AD域的功能采用PAM认证。

- /etc/sssd/sssd.conf
```test
[sssd]
domains = ad001.siemens.net  #名称对应如下配置[domain/ad001.siemens.net] 
config_file_version = 2
services = nss, pam

[domain/ad001.siemens.net] 
ad_server = ad001.siemens.net
ad_domain = ad001.siemens.net
krb5_realm = ad001.siemens.net
realmd_tags = manages-system joined-with-adcli  
cache_credentials = True
id_provider = ad
krb5_store_password_if_offline = True
default_shell = /bin/bash
ldap_id_mapping = True
use_fully_qualified_names = true  #是否启用全名，待产品确认
fallback_homedir = /home/%u@%d # home目录位置，待产品确认%u=ug-ads，%d=域用户名
access_provider = simple
```
- systemctl start sssd


- /etc/krb5.conf
```txt
[logging]        
default = /var/log/krb5libs.log        
kdc = /var/log/krb5kdc.log        
admin_server = /var/log/kadmind.log

[libdefaults]        
default_realm = AD001.SIEMENS.NET        
dns_lookup_realm = true        
dns_lookup_kdc = true        
ticket_lifetime = 24h        
renew_lifetime = 7d        
forwardable = true

# don't set the following parameter because sssd auth will always use the same    
# file for any user as kerberos cache, which will not work because of permissions    
##default_ccache_name = /tmp/krb5cc_0
 
[realms]        
AD001.SIEMENS.NET = {                
kdc = ad001.siemens.net                
admin_server = ad001.siemens.net                     
}

[domain_realm]        
ad001.siemens.net = AD001.SIEMENS.NET        
.ad001.siemens.net = AD001.SIEMENS.NET

[appdefaults]
 forwardable = true
 pam = {
  minimum_uid = 5000
  search_k5login = yes
  # Use "debug = true" to get debug messages of pam_krb module in syslog
 }
```

- /etc/nsswitch.conf
```txt
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.
passwd:         files   sss winbind
group:          files   sss winbind
shadow:         files  sss winbind
gshadow:        files

hosts:          files mdns4_minimal [NOTFOUND=return] dns
networks:       files

protocols:      db files
services:       db files 
ethers:         db files
rpc:            db files

netgroup:       nis sss
automount:      sss
```

- /etc/samba/smb.conf
```txt
include = /etc/samba/smb-ads.conf 新增配置文件smb-ads.conf ：
include = /etc/samba/smb-aaa.conf
smb-aaa.conf
include = /etc/samba/smb-ads.conf
# or include = /etc/samba/smb-ldap.conf


security = ads
dedicated keytab file = /etc/krb5.keytab
kerberos method = secrets and keytab
realm = AD001.SIEMENS.NET 
template shell = /bin/bash
winbind offline logon = true
winbind enum users = Yes
winbind enum groups = Yes
winbind separator = /
idmap config * : range = 1000000-1999999
idmap config * : backend        = tdb
winbind use default domain = yes
winbind use krb5 enterprise principals = yes
winbind scan trusted domains = Yes
```

- systemctl start smbd
- systemctl enable smbd
- systemctl start winbind
- systemctl enable winbind
- su test1@ad001.siemens.net