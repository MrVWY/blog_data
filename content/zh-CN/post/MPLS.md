---
title: "MPLS"
date: "2021-01-28 09:17:08"
categories:
  - 路由交换
tags:
  - 路由交换
toc: true
math: true
type: about
---

### 基本概念

#### 转发等价类

​	MPLS作为一种分类转发技术，将具有相同转发处理方式的分组归为一类，称为FEC（Forwarding Equivalence Class，转发等价类）。相同FEC的分组在MPLS网络中将获得完全相同的处理。

​	FEC的划分方式非常灵活，可以是以源地址、目的地址、源端口、目的端口、协议类型或VPN等为划分依据的任意组合。例如，在传统的采用最长匹配算法的IP转发中，到同一个目的地址的所有报文就是一个FEC。

#### 标签

![](/images/MPLS/1.png)

标签是一个长度固定，仅具有本地意义的短标识符，长度为4个字节，用于唯一标识一个分组所属的FEC。一个标签只能代表一个FEC。

标签共有4个域：

- Label：标签值字段，长度为20bits，用来标识一个FEC。
-  Exp：3bits，保留，协议中没有明确规定，通常用作CoS。
-  S：1bit，MPLS支持多重标签。值为1时表示为最底层标签。
-  TTL：8bits，和IP分组中的TTL意义相同，可以用来防止环路。

#### 标签交换路由器

LSR（Label Switching Router，标签交换路由器）是MPLS网络中的基本元素，所有LSR都支持MPLS技术

#### 标签交换路径

```
                 R1------->R2------->R3
```

一个转发等价类在MPLS网络中经过的路径称为LSP（Label Switched Path，标签交换路径）。在一条LSP上，沿数据传送的方向，相邻的LSR分别称为上游LSR和下游LSR。如上述中，R2为R1的下游LSR，相应的，R1为R2的上游LSR。

#### 标签分发协议

LDP（Label Distribution Protocol，标签分发协议）是MPLS的控制协议，它相当于传统网络中的信令协议，负责FEC的分类、标签的分配以及LSP的建立和维护等一系列操作。

MPLS可以使用多种标签发布协议，包括专为标签发布而制定的协议，例如：LDP、CR-LDP（Constraint-Based Routing using LDP，基于约束路由的LDP）；也包括现有协议扩展后支持标签发布的，例如：BGP（Border Gateway Protocol，边界网关协议）、RSVP（Resource Reservation Protocol，资源预留协议）。同时，还可以手工配置静态LSP。

#### LSP隧道技术

一条LSP的上游LSR和下游LSR，尽管它们之间的路径可能并不在路由协议所提供的路径上，但是MPLS允许在它们之间建立一条新的LSP，这样，上游LSR和下游LSR分别就是这条LSP的起点和终点。这时，上游LSR和下游LSR间的LSP就是LSP隧道，它避免了采用传统的网络层封装隧道。

如果隧道经由的路由与逐跳从路由协议中取得的路由一致，这种隧道就称为`逐跳路由隧道（Hop-by-Hop Routed Tunnel）`；否则称为`显式路由隧道（Explicitly Routed Tunnel）`。

#### 标签栈

如果分组在超过一层的LSP隧道中传送，就会有多层标签，形成标签栈（Label Stack）。在每一隧道的入口和出口处，进行标签的入栈（PUSH）和出栈（POP）操作。

标签栈按照“后进先出”（Last-In-First-Out）方式组织标签，MPLS从栈顶开始处理标签。

MPLS对标签栈的深度没有限制。若一个分组的标签栈深度为m，则位于栈底的标签为1级标签，位于栈顶的标签为m级标签。未压入标签的分组可看作标签栈为空（即标签栈深度为零）的分组

### 结构

#### 网络结构

MPLS网络的基本构成单元是LSR，由LSR构成的网络称为MPLS域。

位于MPLS域边缘、连接其它用户网络的LSR称为LER（Label Edge Router，边缘LSR），区域内部的LSR称为核心LSR。核心LSR可以是支持MPLS的路由器，也可以是由ATM交换机等升级而成的ATM-LSR。域内部的LSR之间使用MPLS通信，MPLS域的边缘由LER与传统IP技术进行适配。

分组在入口LER被压入标签后，沿着由一系列LSR构成的LSP传送，其中，入口LER被称为Ingress，出口LER被称为Egress，中间的节点则称为Transit。

#### 节点结构

![](/images/MPLS/2.png)

MPLS节点由两部分组成：

- 控制平面（Control Plane）：负责标签的分配、路由的选择、标签转发表的建立、标签交换路径的建立、拆除等工作；
- 转发平面（Forwarding Plane）：依据标签转发表对收到的分组进行转发。

对于普通的LSR，在转发平面只需要进行标签分组的转发，需要使用到LFIB（Label Forwarding Information Base，标签转发表）。对于LER，在转发平面不仅需要进行标签分组的转发，也需要进行IP分组的转发，所以既会使用到LFIB，也会使用到FIB（Forwarding Information Base，转发信息表）
