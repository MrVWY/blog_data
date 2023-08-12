---
title: "Linux Route Flags and Types"
date: "2021-04-03 18:51:31"
categories:
  - TCP/IP、Linux
tags:
  - TCP/IP、Linux
toc: true
math: true
type: about
---

#### Route Flags

- U（up）：此路由处于活动（up）中
- G（Gateway）：指向网关
- H（Host）：指向一个主机
- R（Reinstate）：由动态路由重新初始化的路由
- D（Dynamically）：dynamically installed by daemon or redirect
- M（Modified）：modified from routing daemon or redirect
- ！：拒绝路由

#### Route Types

- unicast：单播路由

- broadcast：目的路由是广播地址

- local：目的地已分配给此主机。数据包被环回并在本地传递

- nat：一条特殊的NAT路由。 前缀所覆盖的目的地被认为是虚拟（或外部）地址，在转发之前需要将其转换为真实（或内部）地址。 使用属性via选择要转换为的地址

  ```
  ip route add nat 193.7.255.184 via 172.16.82.184
  ```

- unreachable：目的路由无法到达。丢弃数据包并生成ICMP消息主机不可访问，本地发件人收到一个EHostUnreach 错误。

  ```
  ip route add unreachable 172.16.82.184
  ```

- prohibit： 目的路由无法到达。数据包将被丢弃，并生成管理上禁止的ICMP消息通信。 本地发件人收到EACCES错误

  ```
  ip route add prohibit 10.21.82.157
  ```

- blackhole：路由黑洞（目的路由无法到达。数据包被悄悄丢弃），不发送ICMP也不转发数据包

  ```
  ip route add blackhole 202.143.170.0/24
  ```

- throw：与策略规则一起使用的特殊控制路径。 如果选择了这样的路由，则会在未找到路由的情况下终止此表中的查找。 如果没有策略路由，则等同于路由表中没有路由。 数据包被丢弃，并生成ICMP消息net unreachable。 本地发件人收到EHostUnreach错误

  ```
  ip route add throw 10.79.0.0/16
  ```

  