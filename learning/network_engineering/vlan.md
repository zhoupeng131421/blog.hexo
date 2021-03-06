---
title: vlan 基本知识点
date: 2019-10-31
tags: vlan
categories: [Learning, vlan]
---

# VLAN的访问链接
> 访问链接（Access Link）
> 汇聚链接（Trunk Link）

## 访问链接
> 只属于一个VLAN，且仅向该VLAN转发数据帧的端口，通常连的时客户机

- 设置VLAN顺序：
  - 生成VLAN
  - 设定访问链接（决定各端口属于哪个VLAN）
    - 静态VLAN
    - 动态VLAN

### 静态VLAN
> 基于端口的VLAN，明确指定各端口属于哪个VLAN的设定方法。

- 客户机每次变更所连端口，都必须同时更改该端口所属VLAN的设定——这显然静态VLAN不适合那些需要频繁改变拓补结构的网络。

### 动态VLAN
> 根据每个端口所连的计算机，随时改变端口所属的VLAN

- 分类：
  - 基于MAC地址的VLAN（MAC Based VLAN）
  - 基于子网的VLAN（Subnet Based VLAN）
  - 基于用户的VLAN（User Based VLAN）

1. 基于MAC地址的VLAN，就是通过查询并记录端口所连计算机上网卡的MAC地址来决定端口的所属。
2. 基于子网的VLAN，则是通过所连计算机的IP地址，来决定端口所属VLAN的。
3. 基于用户的VLAN，则是根据交换机各端口所连的计算机上当前登录的用户，来决定该端口属于哪个VLAN。
    - 这里的用户识别信息，一般是计算机操作系统登录的用户，比如可以是Windows域中使用的用户名。这些用户名信息

## 汇聚链接 (设置跨越多台交换机的VLAN)
> 能够转发多个不同VLAN的通信的端口。汇聚链路上流通的数据帧，都被附加了用于识别分属于哪个VLAN的特殊信息。

### 汇聚链接实现
1. A发送的数据帧从交换机1经过汇聚链路到达交换机2时，在数据帧上附加了表示属于红色VLAN的标记。
2. 交换机2收到数据帧后，经过检查VLAN标识发现这个数据帧是属于红色VLAN的。
3. 因此去除标记后根据需要将复原的数据帧只转发给其他属于红色VLAN的端口。
- VLAN识别信息，有可能支持标准的“IEEE 802.1Q”协议，也可能是Cisco产品独有的“ISL（Inter Switch Link）”。如果交换机支持这些规格，那么用户就能够高效率地构筑横跨多台交换机的VLAN。
- 汇聚链路上流通着多个VLAN的数据，自然负载较重。因此，在设定汇聚链接时，有一个前提就是必须支持100Mbps以上的传输速度。
  - 默认条件下，汇聚链接会转发交换机上存在的所有VLAN的数据。换一个角度看，可以认为汇聚链接（端口）同时属于交换机上所有的VLAN。由于实际应用中很可能并不需要转发所有VLAN的数据，因此为了减轻交换机的负载、也为了减少对带宽的浪费，我们可以通过用户设定限制能够经由汇聚链路互联的VLAN。
### 汇聚方式
> IEEE802.1Q and ISL
#### IEEE802.1Q
- IEEE802.1Q所附加的VLAN识别信息，位于数据帧中“发送源MAC地址”与“类别域（Type Field）”之间。具体内容为2字节的TPID和2字节的TCI，共计4字节。
- 在数据帧中添加了4字节的内容，那么CRC值自然也会有所变化。这时数据帧上的CRC是插入TPID、TCI后，对包括它们在内的整个数据帧重新计算后所得的值。
- 而当数据帧离开汇聚链路时，TPID(固定的 0x8100)和TCI会被去除，这时还会进行一次CRC的重新计算。
- 基于IEEE802.1Q附加的VLAN信息，就像在传递物品时附加的标签。因此，它也被称作“标签型VLAN（Tagging VLAN）”。

#### ISL(Inter Switch Link)
> ISL，是Cisco产品支持的一种与IEEE802.1Q类似的、用于在汇聚链路上附加VLAN信息的协议。

- 使用ISL后，每个数据帧头部都会被附加26字节的“ISL包头（ISL Header）”，并且在帧尾带上通过对包括ISL包头在内的整个数据帧进行计算后得到的4字节CRC值。换而言之，就是总共增加了30字节的信息。
- 当数据帧离开汇聚链路时，只要简单地去除ISL包头和新CRC就可以了。由于原先的数据帧及其CRC都被完整保留，因此无需重新计算CRC。
- ISL有如用ISL包头和新CRC将原数据帧整个包裹起来，因此也被称为“封装型VLAN（Encapsulated VLAN）”。
- ISL是Cisco独有的协议，因此只能用于Cisco网络设备之间的互联。

### VLAN间路由
> 路由功能，一般主要由路由器提供。但在今天的局域网里，我们也经常利用带有路由功能的交换机——三层交换机（Layer 3 Switch）来实现。

- 使用支持汇聚链接的路由器（当使用一条网线连接路由器与交换机、进行VLAN间路由时，需要用到汇聚链接。）
- 数据通路： 发送方 -- 交换机 -- 路由器 -- 交换机 -- 接收方
![](./vlan/diff_vlan_com.png)

#### 三层交换机
> 如果使用路由器进行VLAN间路由的话，随着VLAN之间流量的不断增加，很可能导致路由器成为整个网络的瓶颈。

> 交换机使用被称为ASIC（Application Specified Integrated Circuit）的专用硬件芯片处理数据帧的交换操作，在很多机型上都能实现以缆线速度（Wired Speed）交换。而路由器，则基本上是基于软件处理的。

> 就VLAN间路由而言，流量会集中到路由器和交换机互联的汇聚链路部分，这一部分尤其特别容易成为速度瓶颈。

- 三层交换机，本质上就是“带有路由功能的（二层）交换机”
![](./vlan/3layer_switch.png)
- 数据传输流程：发送方 -- 交换模块 -- 路由模块 -- 交换模块 -- 接收方
![](./vlan/interplay_3layer_switch.png)
