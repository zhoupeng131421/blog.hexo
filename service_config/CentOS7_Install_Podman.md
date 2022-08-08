---
title: CentOS7 install podman
date: 2021-9-3
tags: [podman, centos]
categories: 服务配置
---

# Install podman
## install
- sudo yum install -y podman

## config
- mkdirk -p /mydata/lib/containers/{run,graph}
- vim /etc/containers/storage.conf:
    - runroot = "/mydata/lib/containers/run"
    - graphroot = "/mydata/lib/containers/graph"

- which podman
- ln -s /usr/bin/podman /usr/bin/docker

## vefiry
- podman pull centos
- podman images

# podman command:
- podman network create xxx
     - podman network inspect xxx
- podman volume create xxx
    - podman volume ls
    - podman volume inspect xxx
- podman inspect contaimer_name_ID | grep IPAddress
    - 用来显示各个容器的内部ip
- podman pull xxx
- podman info
- 获取容器pid: podman inspect -f {{.State.Pid}} xxx
- 进去容器网络命名空间： nsenter -n -t pid

# Install Nextcloud
## prepare
- mkdir /var/lib/mysql
    - for mariadb storage
- yum install httpd
    - for nextcloud web service
- sudo podman network create nextcloud-net
    - /etc/cni/net.d/nextcloud-net.conflist
- sudo podman network inspect nextcloud-net
    - 在上一个命令中可以看到，您创建了一个名为“dns.podman”的DNS区域。可通过“CONTAINER_NAME.dns.podman”访问此网络中创建的所有容器。
- sudo podman volume create nextcloud-app
- sudo podman volume create nextcloud-data
- sudo podman volume create nextcloud-db
    - sudo podman volume ls
- sudo podman volume inspect nextcloud-app
- podman run -itd --name nextcloud-db -h mysql -v nextcloud-db:/var/lib/mysql -e xxx -e xxx mariadb
    - -i, --interactive: Keep STDIN open even if not attached
    - -t, --tty: Allocate a pseudo-TTY for container
    - -d, --detach: Run container in background and print container ID
    - --name: Assign a name to the container
    - -h: Set container hostname
    - --privileged: Give extended privileges to container
    - -p: Publish a container's port, or a range of ports, to the host (default []). host_port : container_port
    - -v, --volume: Bind mount a volume into the container (default [])
    - -e, --env: Set environment variables in container
## deploy mariadb
- podman pull mariadb
- podman run -itd --env MYSQL_DATABASE=nextcloud --env MYSQL_USER=nextcloud --env MYSQL_PASSWORD=DB_USER_PASSWORD --env MYSQL_ROOT_PASSWORD=DB_ROOT_PASSWORD --volume nextcloud-db:/var/lib/mysql --network nextcloud-net --restart on-failure --name nextcloud-db -h nextcloud-db mariadb
- podman run -itd --env MYSQL_DATABASE=nextcloud --env MYSQL_USER=nextcloud --env MYSQL_PASSWORD=zp@131421 --env MYSQL_ROOT_PASSWORD=zp@131421 --volume nextcloud-db:/var/lib/mysql --network nextcloud-net --restart on-failure --name nextcloud-db mariadb
- podman run -itd --env  MYSQL_ROOT_PASSWORD=xxx --volume nextcloud-db:/var/lib/mysql --network nextcloud-net --restart on-failure --name nextcloud-db --publish 3306:3306 mariadb
- podman run --detach
　--env MYSQL_DATABASE=nextcloud
　--env MYSQL_USER=nextcloud
　--env MYSQL_PASSWORD=DB_USER_PASSWORD
　--env MYSQL_ROOT_PASSWORD=DB_ROOT_PASSWORD
　--volume nextcloud-db:/var/lib/mysql
　--network nextcloud-net
　--restart on-failure
　--name nextcloud-db
　docker.io/library/mariadb:10
- 检查运行中的容器
    - podman container ls
- 数据库数据就存在 /mydata/lib/containers/graph/volumes/nextcloud-db/_data/
- 进入数据库容器： podman exec -it nextcloud-db /bin/bash
    - mysql -u root -p
    - CREATE DATABASE nextcloud;
    - GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextcloud'@'localhost' IDENTIFIED BY 'strong_password';
    - FLUSH PRIVILEGES;
    - exit;
## deploy nextcloud
- podman run --detach
　--env MYSQL_HOST=nextcloud-db.dns.podman
　--env MYSQL_DATABASE=nextcloud
　--env MYSQL_USER=nextcloud
　--env MYSQL_PASSWORD=DB_USER_PASSWORD
　--env NEXTCLOUD_ADMIN_USER=NC_ADMIN
　--env NEXTCLOUD_ADMIN_PASSWORD=NC_PASSWORD
　--volume nextcloud-app:/var/www/html
　--volume nextcloud-data:/var/www/html/data
　--network nextcloud-net
　--restart on-failure
　--name nextcloud
　--publish 8080:80
　docker.io/library/nextcloud:20
- podman run -itd --env MYSQL_HOST=nextcloud-db.dns.podman --env MYSQL_DATABASE=nextcloud --env MYSQL_USER=nextcloud --env MYSQL_PASSWORD=DB_USER_PASSWORD --env NEXTCLOUD_ADMIN_USER=NC_ADMIN --env NEXTCLOUD_ADMIN_PASSWORD=NC_PASSWORD --volume nextcloud-app:/var/www/html --volume nextcloud-data:/var/www/html/data --network nextcloud-net --restart on-failure --name nextcloud --publish 8080:80 nextcloud
- 请注意，应将DB_USER_PASSWORD替换为上一步中使用的密码。将 NC_ADMIN 和 NC_PASSWORD 替换为要用于 Nextcloud 管理员帐户的用户名和密码.
- podman run -itd --env MYSQL_HOST=nextcloud-db.dns.nextcloud-net --env MYSQL_DATABASE=nextcloud --env MYSQL_USER=nextcloud --env MYSQL_PASSWORD=zp@131421 --env NEXTCLOUD_ADMIN_USER=admin --env NEXTCLOUD_ADMIN_PASSWORD=zp@131421 --volume nextcloud-app:/var/www/html --volume nextcloud-data:/var/www/html/data --network nextcloud-net --restart on-failure --name nextcloud --publish 80:80 nextcloud
- 检查运行中的容器
    - podman container ls

##

# Update
## update mariadb
- podman pull mariadb:10
- podman stop nextcloud-db
- podman rm nextcloud-db
- podman run --detach
　--env MYSQL_DATABASE=nextcloud
　--env MYSQL_USER=nextcloud
　--env MYSQL_PASSWORD=DB_USER_PASSWORD
　--env MYSQL_ROOT_PASSWORD=DB_ROOT_PASSWORD
　--volume nextcloud-db:/var/lib/mysql
　--network nextcloud-net
　--restart on-failure
　--name nextcloud-db
　docker.io/library/mariadb:10
## update nextcloud
- podman pull nextcloud:20
- podman stop nextcloud
- podman rm nextcloud
- podman run --detach
　--env MYSQL_HOST=nextcloud-db.dns.podman
　--env MYSQL_DATABASE=nextcloud
　--env MYSQL_USER=nextcloud
　--env MYSQL_PASSWORD=DB_USER_PASSWORD
　--env NEXTCLOUD_ADMIN_USER=NC_ADMIN
　--env NEXTCLOUD_ADMIN_PASSWORD=NC_PASSWORD
　--volume nextcloud-app:/var/www/html
　--volume nextcloud-data:/var/www/html/data
　--network nextcloud-net
　--restart on-failure
　--name nextcloud
　--publish 8080:80
　docker.io/library/nextcloud:20





podman run -itd --env MYSQL_DATABASE=nextcloud --env MYSQL_USER=nextcloud --env MYSQL_PASSWORD=123456 --env MYSQL_ROOT_PASSWORD=123456 --volume nextcloud-db:/var/lib/mysql --restart on-failure --name nextcloud-db --network nextcloud-net --publish 3306:3306 --hostname nextcloud-db mariadb
podman run -itd --env MYSQL_HOST=nextcloud-db.dns.nextcloud-net --env MYSQL_DATABASE=nextcloud --env MYSQL_USER=nextcloud --env MYSQL_PASSWORD=zp@131421 --env NEXTCLOUD_ADMIN_USER=admin --env NEXTCLOUD_ADMIN_PASSWORD=zp@131421 --volume nextcloud-app:/var/www/html --volume nextcloud-data:/var/www/html/data --restart on-failure --name nextcloud --network nextcloud-net --publish 80:80 --hostname nextcloud nextcloud


podman run -itd --env  MYSQL_ROOT_PASSWORD=zp@131421 --volume nextcloud-db:/var/lib/mysql --restart on-failure --name nextcloud-db --publish 3306:3306 mariadb

podman run -itd --env NEXTCLOUD_ADMIN_USER=admin --env NEXTCLOUD_ADMIN_PASSWORD=zp@131421 --volume nextcloud-app:/var/www/html --volume nextcloud-data:/var/www/html/data --restart on-failure --name nextcloud --publish 80:80 nextcloud