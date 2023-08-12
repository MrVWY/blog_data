---
title: "Linux virtual devices"
date: "2021-04-15 15:34:10"
categories:
  - Linux
tags:
  - Linux
toc: true
math: true
type: about
---

#### Linux virtual devices Type

Bridge、Veth、Tun、Tap

#### Linux namespace

namespace的本质就是指：一种在空间上隔离的概念，当下盛行的许多容器虚拟化技术（典型代表如LXC、Docker）就是基于linux命名空间的概念而来的。

其主要包括：

- `Cgroup`（物理资源隔离）
- `IPC`
- `Mount`
- `PID`
- `User`
- `UTS`
- `Network`

##### network namespace

network namespace 能创建多个隔离的网络空间，它们有独自的网络栈信息。不管是虚拟机还是容器，运行的时候仿佛自己就在独立的网络中。

对于不同的网络空间，因为其互相之间隔离，因此可以使用Veth接口来连通不同网络空间

###### use

```
//create ns1 network namespace
ip nets add ns1
ip nets list
//add veth1 interface to ns1 network namespace
ip link set veth1 netns ns1
//add IP in veth1
//ip netns exec <network namespace name> <command>
ip netns exec ns1 ip addr add <IP> dev veth1
//run shell in network namespace
ip netns exec <network namespace name> bash  //back: "exit"
```

#### Bridge

linux的Bridge网桥，其有点类似于交换机，也是在其连接的接口间转发数据包。

#### Veth-Pair

veth-pair 是一对的虚拟设备接口，和 tap/tun 设备不同的是，它都是成对出现的。一端连着协议栈，一端彼此相连着。一旦一端设备关闭，那么另外一端的设备也将会被关闭。

```
//create veth0<-->veth1
ip link add veth0 type veth peer name veth1
```

#### Tun/Tap

Tun/Tap设备可以在协议栈中把数据包转发给上层用户自定义的程序去处理。

##### 流程

```
过程1-2-3-4-5-6-7-8
+----------------------------------------------------------------+
|                                                                |
|  +--------------------+      +--------------------+            |
|  | User Application A |      | User Application B |<-----+     |
|  +--------------------+      +--------------------+      |     |
|               | 1                    | 5                 |     |
|...............|......................|...................|.....|
|               ↓                      ↓                   |     |
|         +----------+           +----------+              |     |
|         | socket A |           | socket B |              |     |
|         +----------+           +----------+              |     |
|                 | 2               | 6                    |     |
|.................|.................|......................|.....|
|                 ↓                 ↓                      |     |
|             +------------------------+                 4 |     |
|             | Newwork Protocol Stack |                   |     |
|             +------------------------+                   |     |
|                | 7                 | 3                   |     |
|................|...................|.....................|.....|
|                ↓                   ↓                     |     |
|        +----------------+    +----------------+          |     |
|        |      eth0      |    |      tun0      |          |     |
|        +----------------+    +----------------+          |     |
|    2.2.2.2     |                   |   1.1.1.1           |     |
|                | 8                 +---------------------+     |
|                |                                               |
+----------------|-----------------------------------------------+
                 ↓
         Physical Network
```

##### 区别

通过Tun设备只能读写IP数据包，而通过Tap设备能读写链路层数据包

##### songgao/water

具体操作方法看官网文档：https://github.com/songgao/water

###### 如何实现

由于本人十分好奇它底层是如何创建Tun/Tap设备，因此走了一轮源码。

```
func ioctl(fd uintptr, request uintptr, argp uintptr) error {
	_, _, errno := syscall.Syscall(syscall.SYS_IOCTL, fd, uintptr(request), argp)
	if errno != 0 {
		return os.NewSyscallError("ioctl", errno)
	}
	return nil
}
//可以发现通过syscall.Syscall发送信号syscall.SYS_IOCTL调用Linux内核函数ioctl来创建Tun/Tap设备
```

#### Reference

https://gist.github.com/mtds/4c4925c2aa022130e4b7c538fdd5a89f