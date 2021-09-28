---
title: kubernetes deploy
date: 2021-3-11
tags: [kubernetes]
categories: Learning
---

- hosts:
    - k8s-master-01
    - k8s-master-02
    - k8s-master-03
    - k8s-node-01
    - k8s-node-02
    - k8s-node-03

# prepare
- prepare-k8s.yml:
```shell
---
- name: prepare kubernetes nodes
  gather_facts: no
  hosts: k8s-master
  tasks:
    - name: Set time zone
      timezone:
        name: Asia/Shanghai
    - name: Update System
      yum:
        name: "*"
        state: latest
        update_cache: yes
    - name: Install common packages
      yum:
        name: [epel-release, conntrack, ntpdate, ntp, ipvsadm, ipset, jq, iptables, curl, sysstat, libseccomp, wget, vim, net-tools, git]
        state: present
        update_cache: yes
    - name: Config Firewall
      shell: |
        systemctl stop firewalld
        systemctl disable firewalld
        yum -y install iptables-services
        systemctl start iptables
        systemctl enable iptables
        iptables -F
        service iptables save
    - name: Disable swap
      shell: |
        swapoff -a
        sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    - name: Disable SeLinux
      lineinfile:
        dest: /etc/selinux/config
        regexp: '^SELINUX='
        line: 'SELINUX=disabled'
```
- modify kernel parameter:
    - vim kubernetes.conf:
    ```shell
    net.bridge.bridge-nf-call-iptables=1
    net.bridge.bridge-nf-call-ip6tables=1
    net.ipv4.ip_forward=1
    net.ipv4.tcp_tw_recycle=0
    vm.swappiness=0
    vm.overcommit_memory=1
    vm.panic_on_oom=0
    fs.inotify.max_user_instances=9182
    fs.inotify.max_user_watches=1048576
    fs.file-max=52706963
    fs.nr_open=52706963
    net.ipv6.conf.all.disable_ipv6=1
    net.netfilter.nf_conntrack_max=2310720
    ```
    - cp kubernetes.conf /etc/sysctl.d/
    - sysctl -p /etc/sysctl.d/kubernetes.conf
        - if error:"bridge-nf-call-iptables: No such file or directory" --> `modprobe br_netfilter`
        - if error:"/proc/sys/net/netfilter/nf_conntrack_max: No such file or directory" --> `modprobe ip_conntrack`
- timezone:
    - timedatectl set-timezone Asia/Shanghai
    - timedatectl set-local-rtc 0
    - systemctl restart rsyslog
    - systemctl restart crond
- close unused sys server
    - systemctl stop postfix
    - systemctl disable postfix
- config rsyslogd and systemd journald
    - mkdir /var/log/journal    # 持久化保存日志
    - mkdir /etc/systemd/journal.conf.d
    - vim /etc/systemd/journal.conf.d/99-prophet.conf:
    ```shell
    [Journal]
    # 持久化保存到磁盘
    Storage=persistent
    # 压缩历史日志
    Compress=yes
    SyncIntervalSec=5m
    RateLimitInterval=30s
    RateLimitBurst=1000
    # 最大占用空间 10G
    SystemMaxUse=10G
    # 单日志文件最大 200M
    SystemMaxFileSize=200M
    # 日志保存时间 2 周
    MaxRetentionSec=2week
    # 不将日志转发到 
    syslogForwardToSyslog=no
    ```
    - systemctl restart systemd-journald
- update kernel
    - rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
    - 安装完成后检查 /boot/grub2/grub.cfg 中对应内核 menuentry 中是否包含 initrd16 配置，如果没有，再安装一次！
    - yum --enablerepo=elrepo-kernel install -y kernel-lt
    - 设置开机从新内核启动
        - grub2-set-default "CentOS Linux (5.4.104-1.el7.elrepo.x86_64) 7 (Core)"
    - 重启后安装内核源文件
        - yum --enablerepo=elrepo-kernel install kernel-lt-devel-$(uname -r) kernel-lt-headers-$(uname -r)
- close NUMA:
    - cp /etc/default/grub{,.bak}
    - vim /etc/default/grub
        - 在 GRUB_CMDLINE_LINUX 一行添加 `numa=off` 参数，如下所示：
        - diff /etc/default/grub.bak /etc/default/grub
        ```shell
        6c6
        < GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rhgb quiet"
        ---
        > GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rhgb quiet numa=off"
        ```
        - cp /boot/efi/EFI/centos/grub.cfg{,.bak}
        - grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg
