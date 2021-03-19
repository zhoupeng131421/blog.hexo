---
title: ceph deploy prepare
date: 2021-3-3
tags: [ceph]
categories: 服务配置
---

> deploy Ceph with cephadm

# prepare
- systemctl disable firewalld
- systemctl stop firewalld
- setenforce 0
- vim /etc/enforce/config
```shell
disabled
```

## install apps
- config proxy: vim ~/.bashrc
```shell
export http_proxy=http://x.x.x.x:x
export https_proxy=http://x.x.x.x:x
```
- sudo yum install net-tools vim wget chrony

## config sudoers
- desc: allow personal user has access for sudo
- `su root`
- `chmod +w /etc/sudoers`
- `vim /etc/sudoers`:

    ```shell
    root    ALL=(ALL:ALL) ALL
    <user>  ALL=(ALL:ALL) NOPASSWD:ALL  #new
    Defaults env_keep += "ftp_proxy http_proxy https_proxy no_proxy"
    ```
- `chmod -w /etc/sudoers`

## install python3
- depended package
    - yum -y groupinstall "Development tools"
    - yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel
- wget https://www.python.org/ftp/python/3.7.9/Python-3.7.9.tar.xz
- sudo mkdir /usr/local/python3
- tar -xvJf  Python-3.7.9.tar.xz
- cd Python-3.7.9
- ./configure --prefix=/usr/local/python3
- make && make install
- if not have the pip, also need install pip: `yum -y install python-pip`
- ln -s /usr/local/python3/bin/python3 /usr/bin/python3
- ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3

## install docker
- uninstall old versions
    - sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
- setup the  repository
    - sudo yum install -y yum-utils
    - sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
- install docker engine:
    - sudo yum install -y docker-ce docker-ce-cli containerd.io
    - sudo systemctl start docker
    - sudo systemctl enable docker
    - verify: sudo docker run hello-world
- docker proxy:
    - sudo mkdir -p /etc/systemd/system/docker.service.d
    - sudo vim /etc/systemd/system/docker.service.d/http-proxy.conf
    ```shell
    [Service]
    Environment="HTTP_PROXY=http://proxy-addr:proxy-port" "HTTPS_PROXY=http://proxy-addr:proxy-port" "NO_PROXY=localhost,127.0.0.1"
    ```
    - systemctl daemon-reload
    - systemctl restart docker

# install cephadm
- get cephadm cript:
    - curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
- chmod +x cephadm
- sudo mkdir /etc/ceph
- ./cephadm add-repo --release octopus
- ./cephadm install
- deploy the firest ceph node: cephadm bootstrap --mon-ip xx.xx.xx.xx
- enable ceph cli
    - sudo cephadm install ceph-common
    - verify:
        - ceph -v
        - ceph status
- deploy OSDs:
    - ceph orch device ls