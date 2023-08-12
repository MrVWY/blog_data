---
title: "iptables 四表五链"
date: "2021-02-20 00:37:21"
categories:
  - network
tags:
  - network
toc: true
math: true
type: about
---

### 前言

iptables只是一个工具，实际上内部核心是基于[Netfilter](https://zh.wikipedia.org/wiki/Netfilter)框架去实现其功能的一个命令行工具。

### 四表和五链

#### 四表

首先表这概念主要是把相同功能的规则集中在一起。而不是每条规则都散落和重复的出现在每个不同的链中。

##### raw表

raw表主要用来决定是否跟踪连接和配置notrack目标。优先级最高。用法：可以配置该表来决定哪一条连接可以被跟踪。其提供两条调用链来供使用：prerouting、output。 例子：

//对来自于ip为192.168.10.117的所有数据包不进行追踪
iptables -t raw -A PREROUTING -s 192.168.10.117 -j NOTRACK
//对来自于ip为192.168.10.117的所有数据包进行追踪
iptables -t raw -A PREROUTING -s 192.168.10.117 -j TRACK

##### mangle表

该表主要用于数据包的修改，例如：TTL，IP数据包首部，给数据包打上标签(--set-mask)。其提供五条调用链来供使用：prerouting、foward、postrouting、input、output。 参考：[what-is-the-mangle-table-in-iptables](https://serverfault.com/questions/467756/what-is-the-mangle-table-in-iptables)

##### nat表

该表主要用于网络地址转换。其提供4条调用链来供使用：prerouting、postrouting、input、output。

##### filter表

该表是用来过滤网络上的数据包，决定数据包是否可以进入上层协议栈进行处理。其提供3条调用链来供使用：input、output、foward。

#### 五个规则链

链可以看出一道道的“关卡”

##### prerouting链

路由前链，处理刚到达本机的数据包，进行路由决策。可以使用的表：raw、mangle、nat。

##### foward链

转发链，当prerouting链判断目的地不是本机时，会经过该链并执行规则。可以使用的表：mangle、filter。

##### postrouting链

数据包经过处理后，在网卡前，再对其进行规则匹配和修改。可以使用的表：mangle、nat。

##### input链

经prerouting链进行路由决策之后，进入本机所要匹配的规则和修改。可以使用的表：mangle、nat、filter。

##### output链

经过本机处理好的数据并需要向外发送，经该关卡所要匹配的规则和修改。可以使用的表：raw、mangle、nat、filter。

#### 思维图

![](iptables-四表五链/20200914163440890.png)

### 第五张表---security表

该表是用来强制性访问控制(Mandatory Access Control,MAC)，通过设置内核中SElinux来实现。其提供2条调用链来供使用：input、output。不过我记得在某些场景下，这个SElinux通常需要给关闭。因此该表可能在平常用的不多。 ![](iptables-四表五链/20200915105245370.png)

### 工作流程（重）

![](iptables-四表五链/20200914163703653.png)  

### extend module

- connlimit module

  限制每个IP地址同时链接到server端的数量

  `iptables -t filter -I INPUT -p tcp --dprot 22 -m connlimit --connlimit-above 2 -j REJECT `

  `iptables -t filter -I INPUT -p tcp --dprot 22 -m connlimit ! --connlimit-above 2 -j REJECT `

- limit module

  限速

  `iptables -t filter -I INPUT -p icmp -m limit --limit 10/minite -j ACCEPT`

- iprange module

  指定一段连续的IP地址范围，用于匹配package src or des

  `iptables -t filter -I INPUT -m iprange --src-range 192.168.1.127-192.168.1.146 -j DROP`

### cmd  and example

1. forward packages

   查看server 是否开启转发

   ​	`sysctl net.ipv4.ip_forward`

   ​	已开启：`net.ipv4.ip_forward = 1`

   加入规则：`iptables -t nat -A PREROUTING -p tcp -m tcp --dport 8080 -j DNAT --to-destination [destination IP]`

#### parameter

- `-t`：select iptables tables
- `-I`：select iptables lists
- `-o`：use to select which network adapter to out
- `-p`：use to match the type of package 
- `-m`：use tcp extend module
- `-s`：match  src ip
- `-sport -dport`：src and des ip match

### Reference

*   [iptables-Aarchlinux](https://wiki.archlinux.org/index.php/Iptables_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E6%A8%A1%E5%9D%97_%EF%BC%88Modules%EF%BC%89)
*   [netfilter/iptables: why not using the raw table?](https://unix.stackexchange.com/questions/243079/netfilter-iptables-why-not-using-the-raw-table)
*   [iptables(8) - Linux man page](https://linux.die.net/man/8/iptables)
*   [A Deep Dive into Iptables and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)
*   [Netfilter](https://zh.wikipedia.org/wiki/Netfilter)