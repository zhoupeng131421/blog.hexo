---
title: sysctl configuration
date: 2022-09-15
tags: [sysctl]
categories: Learning
---

> 由于默认的Linux内核参数考虑的是最通用的场景,所以需要修改Linux参数,来适用用户特殊的需求. `/proc/sys`目录下存放着大多数内核参数，并且可以在系统运行时进行更改，不过重新启动机器就会失效。`/etc/sysctl.conf`是一个允许改变正在运行中的Linux系统的接口，它包含一些TCP/IP堆栈和虚拟内存系统的高级选项，修改内核参数永久生效。也就是说/proc/sys下内核文件与配置文件`sysctl.conf`中变量存在着对应关系。

- `swapoff -a && swapon -a && sysctl -p` 该命令可以使sysctl.conf中的修改立即生效(除了一些需要重启才生效的，eg: iommu...).

- 一些常用配置:
```shell
#CTCDN系统优化参数

#关闭ipv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
#决定检查过期多久邻居条目
net.ipv4.neigh.default.gc_stale_time=120
#使用arp_announce / arp_ignore解决ARP映射问题
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.all.arp_announce=2
net.ipv4.conf.lo.arp_announce=2
# 避免放大攻击
net.ipv4.icmp_echo_ignore_broadcasts = 1
# 开启恶意icmp错误消息保护
net.ipv4.icmp_ignore_bogus_error_responses = 1
#关闭路由转发
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
#开启反向路径过滤
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
#处理无源路由的包
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
#关闭sysrq功能
kernel.sysrq = 0
#core文件名中添加pid作为扩展名
kernel.core_uses_pid = 1
# 开启SYN洪水攻击保护
net.ipv4.tcp_syncookies = 1
#修改消息队列长度
kernel.msgmnb = 65536
kernel.msgmax = 65536
#设置最大内存共享段大小bytes
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
#timewait的数量，默认180000
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 4096        87380   4194304
net.ipv4.tcp_wmem = 4096        16384   4194304
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
#每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目
net.core.netdev_max_backlog = 262144
#限制仅仅是为了防止简单的DoS 攻击
net.ipv4.tcp_max_orphans = 3276800
#未收到客户端确认信息的连接请求的最大值
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_timestamps = 0
#内核放弃建立连接之前发送SYNACK 包的数量
net.ipv4.tcp_synack_retries = 1
#内核放弃建立连接之前发送SYN 包的数量
net.ipv4.tcp_syn_retries = 1
#启用timewait 快速回收
net.ipv4.tcp_tw_recycle = 1
#开启重用。允许将TIME-WAIT sockets 重新用于新的TCP 连接
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_fin_timeout = 1
#当keepalive 起用的时候，TCP 发送keepalive 消息的频度。缺省是2 小时
net.ipv4.tcp_keepalive_time = 1800
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 15
#允许系统打开的端口范围
net.ipv4.ip_local_port_range = 1024    65000
#修改防火墙表大小，默认65536
net.netfilter.nf_conntrack_max=655350
net.netfilter.nf_conntrack_tcp_timeout_established=1200
# 确保无人能修改路由表
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
```

- the different `all` and `default` in `net.ipv{4, 6}.conf.{all, default, xxx}.xxx`:
    - reference: [What is the difference between "all", "default" and "eth*" in](https://unix.stackexchange.com/questions/90443/what-is-the-difference-between-all-default-and-eth-in-proc-sys-net-ipv#:~:text=In%20sysctl%2C%20the%20%2Fproc%2Fsys%2Fnet%2Fipv%20%2Fconf%2F%20keys%20have%20the,insgesamt%200%20dr-xr-xr-x%200%20root%20root%200%2012.)
    - `default`: it's applied to new interfaces only.
        - For a `host's interfaces`, the default setting will only apply when new devices are created, and host interfaces are traditionally only created at boot time. Therefore, if you simply change the default parameter, the value isn't applied to the host interfaces until the host is rebooted.
        - The behavior is different for `containers, Kubernetes and CNI`. New container interfaces are created all the time. Therefore if you simply change the default parameter it will apply to new interfaces created by new containers, but not for existing container interfaces and not for host interfaces. This can be very confusing!
    - `all`: `all` means to apply to all existing interfaces. The `xxx` behavior for this feature is documented in the Kernel documentation, which you can find at [ip-sysctl.txt](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt). New setting allways gets applied in addition to the current setting, but it depends on the exact setting, if its ORed, ANDed, or whatevered: (see include/linux/inetdevice.h:83 for IN_DEV_{AND,OR,MAX}CONF).