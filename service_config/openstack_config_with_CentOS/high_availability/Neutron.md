---
title: Neutron
date: 2021-4-7
tags: [Neutron]
categories: Learning
---

> Neutron 允许 OpenStack 用户创建和管理网络对象，如网络、子网、端口和路由等，而这些网络对象正是其他 OpenStack 服务正常运行所需的网络资源。

# concept
1. API Server
    - API Server 是 Neutron 的控制器，`用户对网络资源的请求全部先交由 API Server 处理`。API Server 接受到请求后，会调用后端插件进行具体的任务实现。主要实现对L2网络、IP地址管理(IPAM, IP address managment)、L3扩展(L3主要用于对L2网络进行路由，网关用于外网访问)等支持。
    - Neutron包含很多网络插件，主要实现对各种开源虚拟网络支持，对各种商业网络设备和技术的支持，如路由器、物理交换机、虚拟交换机和软件定义网络控制器等。
2. 网络插件和代理
    -  插件与代理的主要功能在于`实现由 API Server 转发的网络资源请求`，如网络端口的插拔、路由增删、络和子网创建以及提供IP地址等。
    - Neutron项目默认自带多个虚拟和物理网络设备插件和代理，如思科的虚拟和物理交换机、NEC的Open flow、Open vSwitch、Linux Bridge、VMware NSX等。
3. Flat网络
    - Neutron 提供两种网络类型，租户网络（Project Network）和供应商网络（Provider Network）。租户网络中每个租户可以创建多个私有网络，租户可自定义私有网络的IP地址范围，不同的租户可以同时使用相同的 IP地址或地址段。与租户网络不同，供应商网络由云管理员创建，并且必须与现有的物理网络匹配。
    - 为了`实现不同租户网络互相隔离`，Neutron 提供了几种不同的网络拓扑与隔离技术，Flat 网络便是其中之一。