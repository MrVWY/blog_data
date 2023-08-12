---
title: "RIB and FIB"
date: "2021-03-04 13:53:11"
categories:
  - 路由交换
tags:
  - 路由交换
toc: true
math: true
type: about
---

### RIB

RIB（ Routing Information Base）是由节点上的各种路由过程建立的，其维护每种协议 ( 包含来自路由协议，如OSPF、is-is、BGP、静态条目 ) 的网络拓扑和路由表，包括许多到达相同目的地前缀的路由

### FIB

FIB （forwarding information base) 是由RIB中的所以路由里选择最佳路由，将它放进FIB中再对数据包按路由进行转发

```
//原文
FIBs are the best route from the possibly many protocols in the RIBs pushed down to fast forwarding lookup memory for the best path(s).
```

###  Gobgpd、zebra(Quagga) and linux kernel routed

![](/images/RIBandFIB/202003011111.png)
### Reference

- [RIBs and FIBs (aka IP Routing Table and CEF Table)](https://blog.ipspace.net/2010/09/ribs-and-fibs.html)
- [Juniper and Cisco Comparisons of RIB, LIB, FIB and LFIB Tables](http://networkstatic.net/juniper-and-cisco-comparisons-of-rib-lib-fib-and-lfib-tables/)
- [Router table and forwarding table [closed]](https://networkengineering.stackexchange.com/questions/18115/router-table-and-forwarding-table/18116#18116)
- https://www.slideshare.net/shusugimoto1986/tutorial-using-gobgp-as-an-ixp-connecting-router
- [gobgp configure zebra](https://github.com/osrg/gobgp/blob/master/docs/sources/zebra.md)

