---
title: CentOS7 samba winbind
date: 2021-11-9
tags: [samba, centos]
categories: 服务配置
---

- sudo yum install pam_krb5* krb5-libs* krb5-workstation* krb5-devel* krb5-auth samba samba-winbind* samba-client* samba-swat* -y
- sudo chkconfig smb on && chkconfig winbind on

- sudo yum install epel-release -y
- sudo yum install msktutil bind-utils -y

- vim /etc/krb5.conf:
```shell
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

- kinit <gid>
- #kinit -k <hostname>
- msktutil --create
