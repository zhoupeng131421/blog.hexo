---
title: CentOS config openLDAP server
date: 2020-12-23
tags: [ubuntu, LDAP]
categories: 服务配置
---

# Prepare
- close SELinux:
    - `setenforce 0`
    - `vim /etc/selinux/config`: SELINUX=disabled
- close firewall:
    - `systemctl stop firewalld.service`
    - `systemctl disable firewalld.service`
- install openldap:
    - `yum install -y openldap openldap-clients openldap-servers`
    - `systemctl start slapd`
    - `systemctl enable slapd`
- install PHP5:
    - phpldapadmin only support PHP5
    - `rpm -Uvh https://mirror.webtatic.com/yum/el7/epel-release.rpm`
    - `rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm`
    - `yum install php55w.x86_64 php55w-cli.x86_64 php55w-common.x86_64 php55w-gd.x86_64 php55w-ldap.x86_64 php55w-mbstring.x86_64 php55w-mcrypt.x86_64 php55w-mysql.x86_64 php55w-pdo.x86_64 --skip-broken`
- install phpldapadmin:
    - `yum install phpldapadmin`

# openLDAP config
- generate a administrator password: `slappasswd -s 123456`
    - mayby output is: {SSHA}LSgYPTUW4zjGtIVtuZ8cRUqqFRv1tWpE
- new config file for change password
    - `cd ~`
    - `vim changepwd.ldif`
    ```shell
        dn: olcDatabase={0}config,cn=config
        changetype: modify
        add: olcRootPW
        olcRootPW: {SSHA}LSgYPTUW4zjGtIVtuZ8cRUqqFRv1tWpE
    ```
    - `ldapadd -Y EXTERNAL -H ldapi:/// -f changepwd.ldif`
- import config:
```shell
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/collective.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/corba.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/duaconf.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/dyngroup.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/java.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/misc.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openldap.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/pmi.ldif
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/ppolicy.ldif
```
- new config file for change domain name:
    - my domain name is benjo.com, administrator account is admin
    - `vim changedomain.ldif`
    ```shell
        dn: olcDatabase={1}monitor,cn=config
        changetype: modify
        replace: olcAccess
        olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=admin,dc=benjo,dc=com" read by * none

        dn: olcDatabase={2}hdb,cn=config
        changetype: modify
        replace: olcSuffix
        olcSuffix: dc=benjo,dc=com

        dn: olcDatabase={2}hdb,cn=config
        changetype: modify
        replace: olcRootDN
        olcRootDN: cn=admin,dc=benjo,dc=com

        dn: olcDatabase={2}hdb,cn=config
        changetype: modify
        replace: olcRootPW
        olcRootPW: {SSHA}LSgYPTUW4zjGtIVtuZ8cRUqqFRv1tWpE

        dn: olcDatabase={2}hdb,cn=config
        changetype: modify
        add: olcAccess
        olcAccess: {0}to attrs=userPassword,shadowLastChange by dn="cn=admin,dc=benjo,dc=com" write by anonymous auth by self write by * none
        olcAccess: {1}to dn.base="" by * read
        olcAccess: {2}to * by dn="cn=admin,dc=benjo,dc=com" write by * read
    ```
    - `ldapmodify -Y EXTERNAL -H ldapi:/// -f changedomain.ldif`
- new config file for memberof func:
    - add-memberof.ldif refint1.ldif refint2.ldif
    - `vim add-memberof.ldif`
    ```shell
        dn: cn=module{0},cn=config
        cn: modulle{0}
        objectClass: olcModuleList
        objectclass: top
        olcModuleload: memberof.la
        olcModulePath: /usr/lib64/openldap

        dn: olcOverlay={0}memberof,olcDatabase={2}hdb,cn=config
        objectClass: olcConfig
        objectClass: olcMemberOf
        objectClass: olcOverlayConfig
        objectClass: top
        olcOverlay: memberof
        olcMemberOfDangling: ignore
        olcMemberOfRefInt: TRUE
        olcMemberOfGroupOC: groupOfUniqueNames
        olcMemberOfMemberAD: uniqueMember
        olcMemberOfMemberOfAD: memberOf
    ```
    - `vim refint1.ldif`:
    ```shell
        dn: cn=module{0},cn=config
        add: olcmoduleload
        olcmoduleload: refint
    ```
    - `vim refint2.ldif`:
    ```shell
        dn: olcOverlay=refint,olcDatabase={2}hdb,cn=config
        objectClass: olcConfig
        objectClass: olcOverlayConfig
        objectClass: olcRefintConfig
        objectClass: top
        olcOverlay: refint
        olcRefintAttribute: memberof uniqueMember  manager owner
    ```
    - `ldapadd -Q -Y EXTERNAL -H ldapi:/// -f add-memberof.ldif`
    - `ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f refint1.ldif`
    - `ldapadd -Q -Y EXTERNAL -H ldapi:/// -f refint2.ldif`
- new config file for generate group 'benjo Company'
    - `vim base.ldif`:
    ```shell
        dn: dc=benjo,dc=com
        objectClass: top
        objectClass: dcObject
        objectClass: organization
        o: benjo Company
        dc: benjo

        dn: cn=admin,dc=benjo,dc=com
        objectClass: organizationalRole
        cn: admin

        dn: ou=People,dc=benjo,dc=com
        objectClass: organizationalUnit
        ou: People

        dn: ou=Group,dc=benjo,dc=com
        objectClass: organizationalRole
        cn: Group
    ```
    - ldapadd -x -D cn=admin,dc=benjo,dc=com -W -f base.ldif

# PHP config
- `vim /etc/httpd/conf.d/phpldapadmin.conf`:
```shell
  <IfModule mod_authz_core.c>
    # Apache 2.4
    Require all granted //only change this line
  </IfModule>
```
- `vim /etc/phpldapadmin/config.php`:
```shell
# line: 398, change uid to cn
$servers->setValue('login','attr','cn');

# line: 460, No anonymous login
$servers->setValue('login','anon_bind',false);

# line: 519, use cn and sn to assure account uniqueness
$servers->setValue('unique','attrs',array('mail','uid','uidNumber','cn','sn'));
```
- `systemctl start httpd`
- `systemctl enable httpd`

# Verify
- http://xx.xx.xx.xx/ldapadmin
- user: admin
- password: 123456