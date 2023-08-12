---
title: "EVPN"
date: "2021-05-16 16:36:37"
categories:
  - 路由交换
tags:
  - 路由交换
toc: true
math: true
type: about
---

最初的VXLAN方案（RFC7348）中没有定义控制平面，是手工配置VXLAN隧道，然后通过流量泛洪的方式进行主机地址的学习。这种方式实现上较为简单，但是会导致网络中存在很多泛洪流量、网络扩展起来困难。

为了解决上述问题，VXLAN引入了EVPN（Ethernet VPN）作为VXLAN的控制平面。EVPN参考了BGP/MPLS IP VPN的机制，通过扩展BGP协议新定义了几种BGP EVPN路由，通过在网络中发布路由来实现VTEP的自动发现、主机地址学习。

采用EVPN作为控制平面具有以下一些优势：

- 可实现VTEP自动发现、VXLAN隧道自动建立，从而降低网络部署、扩展的难度。
- EVPN可以同时发布二层MAC和三层路由信息。

- 可以减少网络中泛洪流量。

### EVPN网络模型

#### EVPN的典型网络模型中包括如下几部分

- VTEP（VXLAN Tunnel End Point，VXLAN隧道端点）：EVPN的边缘设备。EVPN的相关处理都在VTEP上进行。VTEP可以是一台独立的物理设备，也可以是虚拟机所在的服务器。
- VXLAN隧道：两个VTEP之间的点到点逻辑隧道。VTEP为数据帧封装VXLAN头、UDP头和IP头后，通过VXLAN隧道将封装后的报文转发给远端VTEP，远端VTEP对其进行解封装。
- 核心设备：IP核心网络中的设备。核心设备不参与EVPN处理，仅需要根据封装后报文的外层目的IP地址对报文进行三层转发。
- VXLAN网络/EVPN实例：用户网络可能包括分布在不同地理位置的多个站点内的虚拟机。在骨干网上可以利用VXLAN隧道将这些站点连接起来，为用户提供一个逻辑的二层VPN。这个二层VPN称为一个VXLAN网络，也称为EVPN实例。VXLAN网络通过VXLAN ID来标识，VXLAN ID又称VNI（VXLAN Network Identifier，VXLAN网络标识符），其长度为24比特。不同VXLAN网络中的虚拟机不能二层互通。
- VSI（Virtual Switch Instance，虚拟交换实例）：VTEP上为一个VXLAN提供二层交换服务的虚拟交换实例。VSI可以看做是VTEP上的一台基于VXLAN进行二层转发的虚拟交换机。VSI与VXLAN一一对应。
- ES（Ethernet Segment，以太网段）：用户站点连接到VTEP的链路，通过ESI（Ethernet Segment Identifier，以太网段标识符）唯一标识。目前，一个用户站点只能通过一条链路连接一台VTEP，该ES的ESI为0。

### 多归属站点

#### DF选举

当一个站点连接到多台VTEP时，为了避免冗余备份组中的VTEP均发送泛洪流量给该站点，需要在冗余备份组中为每个AC选举一个VTEP作为DF（Designated Forwarder，指定转发者），负责将泛洪流量转发给该AC。其他VTEP作为该AC的BDF（Backup DF，备份DF），不会向本地站点转发泛洪流量。多归属成员通过发送以太网段路由，向远端VTEP通告ES及其连接的VTEP信息，远端VTEP根据ES、VTEP信息选举出DF。

#### 水平分割

在多归属站点组网中，VTEP接收到站点发送的组播、广播和未知单播数据帧后，判断数据帧所属的VXLAN，通过该VXLAN内除接收接口外的所有本地接口和VXLAN隧道转发该数据帧。同一冗余备份组中的VTEP接收到该数据帧后会在本地所属的VXLAN内泛洪，这样数据帧会通过AC泛洪到本地站点，造成环路和站点的重复接收。EVPN通过水平分割解决该问题。水平分割的机制为：VTEP接收到同一冗余备份组中成员转发的广播、组播、未知单播数据帧后，不向具有相同ESI标识的ES转发该数据帧。

### ARP泛洪抑制（代答）

为了避免广播发送的ARP请求报文占用核心网络带宽，VTEP会根据接收到的ARP请求和ARP应答报文、BGP EVPN路由在本地建立ARP泛洪抑制表项。后续当VTEP收到本站点内虚拟机请求其它虚拟机MAC地址的ARP请求时，优先根据ARP泛洪抑制表项进行代答。如果没有对应的表项，则通过VXLAN隧道将ARP请求泛洪到其他站点。ARP泛洪抑制功能可以大大减少ARP泛洪的次数。


![](/images/EVPN/ARP泛洪抑制示意图.png)
ARP泛洪抑制的处理过程如下：

- 虚拟机VM 1发送ARP请求，获取VM 7的MAC地址。
- VTEP 1根据接收到的ARP请求，建立VM 1的ARP泛洪抑制表项，在VXLAN内泛洪该ARP请求。VTEP 1还会通过BGP EVPN将该表项同步给VTEP 2和VTEP 3。
- 远端VTEP（VTEP 2和VTEP 3）解封装VXLAN报文，获取原始的ARP请求报文后，在本地站点的指定VXLAN内泛洪该ARP请求。
- VM 7接收到ARP请求后，回复ARP应答报文。
- VTEP 2接收到ARP应答后，建立VM 7的ARP泛洪抑制表项，通过VXLAN隧道将ARP应答发送给VTEP 1。VTEP 2通过BGP EVPN将该表项同步给VTEP 1和VTEP 3。
- VTEP 1解封装VXLAN报文，获取原始的ARP应答，将ARP应答报文发送给VM 1。
- 在VTEP 1上建立ARP泛洪抑制表项后，虚拟机VM 4发送ARP请求，获取VM 1的MAC地址。
- VTEP 1接收到ARP请求后，建立VM 4的ARP泛洪抑制表项，并查找本地ARP泛洪抑制表项，根据已有的表项回复ARP应答报文，不会对ARP请求进行泛洪。
- 虚拟机VM 10发送ARP请求，获取VM 1的MAC地址。
- VTEP 3接收到ARP请求后，建立VM 10的ARP泛洪抑制表项，并查找ARP泛洪抑制表项，根据已有的表项（VTEP 1通过BGP EVPN同步回复ARP应答报文，不会对ARP请求进行泛洪。