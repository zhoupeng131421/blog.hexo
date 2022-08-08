---
title: CentOS7 config zabbix5
date: 2021-3-10
tags: [zabbix, centos]
categories: 服务配置
---

reference: https://www.cnblogs.com/opsprobe/p/10617500.html

# install
1. 安装zabbix rpm 源
    - rpm -Uvh https://mirrors.aliyun.com/zabbix/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm
    - yum clean all
2. 安装 zabbix server 和 agent
    - yum install zabbix-server-mysql zabbix-agent -y
3. 安装 Software Collections，便于后续安装高版本的 php
    - yum install centos-release-scl -y
4. 启用 zabbix 前端源，修改vi /etc/yum.repos.d/zabbix.repo，将[zabbix-frontend]下的 enabled 改为 1
5. 安装 zabbix 前端和相关环境
    - yum install zabbix-web-mysql-scl zabbix-apache-conf-scl -y
6. yum 安装 centos7 默认的 mariadb 数据库
    - yum install mariadb-server -y
7. 启动数据库，并配置开机自动启动
    - systemctl enable --now mariadb
8. 初始化 mariadb 配置 root 密码
    - mysql_secure_installation
9. root 用户进入 mysql，并建立 zabbix 数据库
```shell
mysql -u root -p

MariaDB [(none)]> create database zabbix character set utf8 collate utf8_bin;

MariaDB [(none)]> create user zabbix@localhost identified by 'zabbix';

MariaDB [(none)]> grant all privileges on zabbix.* to zabbix@localhost;

quit
```
10. 导入 zabbix 数据库，zabbix 数据库用户为 zabbix，密码为 zabbix
    - zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix
        - zabbix
11. zabbix server 配置文件:
    - vim /etc/zabbix/zabbix_server.conf: DBPassword=zabbix
12. vim /etc/opt/rh/rh-php72/php-fpm.d/zabbix.conf:
    - php_value[date.timezone] = Asia/Shanghai
13. 启动相关服务，并配置开机自动启动
    - systemctl restart zabbix-server zabbix-agent httpd rh-php72-php-fpm
    - systemctl enable zabbix-server zabbix-agent httpd rh-php72-php-fpm

# verify
1. 登陆http://ip/zabbix
    - 一直下一步，输入zabbix数据库密码。
    - 最后登陆账号为Admin、zabbix
2. 修改语言为中文。
