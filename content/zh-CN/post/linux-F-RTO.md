---
title: "linux F-RTO"
date: "2022-01-28 11:51:01"
categories:
  - TCP/IP
tags:
  - TCP/IP
toc: true
math: true
type: about
---

F-RTO 全称 Forward RTO-Recovery，一种检测TCP的虚假超时重传的算法。目的是判断RTO是否正常，从而决定是否执行拥塞避免算法

```
rfc5682:
   It has been pointed out that the retransmission timer can expire
   spuriously and cause unnecessary retransmissions when no segments
   have been lost [LK00, GL02, LM03].  After a spurious retransmission
   timeout, the late acknowledgments of the original segments arrive at
   the sender, usually triggering unnecessary retransmissions of a whole
   window of segments during the RTO recovery.  Furthermore, after a
   spurious retransmission timeout, a conventional TCP sender increases
   the congestion window on each late acknowledgment in slow start.
   This injects a large number of data segments into the network within
   one round-trip time, thus violating the packet conservation principle
   [Jac88].
```

造成虚假超时的原因，在rfc5682上指出了3点原因

- 突发的网络拥塞造成RTT增加，触发RTO机制

  ```
  First, some mobile networking technologies involve sudden
  delay spikes on transmission because of actions taken during a hand-
  off.
  ```

- 从低延迟链路进入到高延迟链路，比如路由重收敛（网络的拓扑结构发生变化后，路由表重新建立到发送再到学习直至稳定，并通告网络中所有相关路由器都得知该变化的过程）

  ```
  Second, a hand-off may take place from a low latency path to a
  high latency path, suddenly increasing the round-trip time beyond the
  current RTO value. 
  ```

- 在低延迟链路来了一股竞争流量

  ```
  Third, on a low-bandwidth link the arrival of
  competing traffic (possibly with higher priority), or some other
  change in available bandwidth, can cause a sudden increase of the
  round-trip time.
  ```

造成虚假RTO的后果

​	虚假的RTO会触发RTO机制，利用拥塞避免、快速重传或者快速恢复算法去重传自以为丢失的数据包，但是最终都会向网络中注入重复而无效的数据包，造成网络拥塞。

当发生重传时，先重传还未被ACK的第一个包，如果重传后收到的第一个ACK不是重复ACK，则尝试发送一个之前从未发送过的数据包，等待发出后的第一个ACK，如果这个ACK也不是重复ACK，说明这个重传是虚假重传。

上述第一个不重复的ACK有两个可能，一种是重传出去的包被ACK了，这种情况下不是虚假重传，另一种是原始数据包被ACK了，这种情况是虚假重传，因此收到这个ACK后再发一个未发送过的数据包，如果又收到一个不重复的ACK，说明当前网络情况正常，刚才的重传是虚假重传。

#### Reference

- https://datatracker.ietf.org/doc/html/rfc5682

