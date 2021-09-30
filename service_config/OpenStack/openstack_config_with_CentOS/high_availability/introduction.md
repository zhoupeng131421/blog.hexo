---
title: OpenStack HA introduction
date: 2021-3-15
tags: [OpenStack, HA]
categories: 服务配置
---

- [reference]:https://docs.openstack.org/ha-guide

- openstack 中需要冗余的组件：
    - Network components, such as switches and routers
    - Applications and automatic service migration
    - Storage components
    - Facility services such as power, air conditioning, and fire protection
- 服务类型：
    - 无状态服务： 可在提出请求后提供响应，然后无需进一步关注。为了使无状态服务高度可用，需要提供冗余节点并对其进行负载。包括nova-api， nova-conductor，glance-api，keystone-api，neutron-api，nova-scheduler。
    - 有状态服务： 对服务的后续请求取决于第一个请求的结果。有状态服务更难管理，因为单个动作通常涉及多个请求。使状态服务高度可用可能取决于您选择主动/被动配置还是主动/主动配置。包括OpenStack的数据库和消息队列。
- 有状态服务高可用类型有:
    - active/passive: 通常需要一个VIP处理请求。独立应用（Pacemarkr or Corosync）监督这些服务
    - active/active: 通常无状态服务，通过负载均衡（HAproxy）以及VIP处理请求
- 集群：
    - 高可用环境中每个集群都应该具有奇数个节点，仲裁定义为节点的一半以上。一般都是3 5 ...
- 高可用常用的手段：
    - hardware：
        - redundant switchs
        - bonded interfaces: 多网口
        - load balancers：物理负载均衡器--特殊的路由
        - storage
    - software：
        - HAproxy: http reverse proxy and load balancer for TCP or HTTP applications.
        - keepalived: a routing software that provides facilities for load balancing and high-availability to Linux system and Linux based infrastructures.
        - Pacemaker: Pacemaker cluster stack is a state-of-the-art high availability and load balancing stack for the Linux platform. Pacemaker is used to make OpenStack infrastructure highly available.
- Openstack 中高可用：
    - Openstack components:
        - Openstack APIs: pytyon实现的无状态服务，A/P即可
        - SQL relational database server：
        - Advanced Message Queuing Protocol (AMQP). openstack 内部状态通讯服务
    - 无状态服务：
        - nova-api, nova-conductor, glance-api, keystione-api, neutron-api, nova-scheduler.
    - 有状态服务：
        - OpenStack database, message queue.
    - 集群管理：
        - Awareness of other applications in the stack
        - Awareness of instances on other machines
        - A shared implementation and calculation of quorum
        - Data integrity through fencing (a non-responsive process does not imply it is not doing anything)
        - Automated recovery of failed instances
    - memcached:
        - `Memcached_servers = controller1:11211,controller2:11211,controller3:11211`
        - By default, controller1 handles the caching service. If the host goes down, controller2 or controller3 will complete the service
