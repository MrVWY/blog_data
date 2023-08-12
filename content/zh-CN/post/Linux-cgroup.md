---
title: "linux cgroup"
date: "2021-04-13 15:34:10"
categories:
  - Linux
tags:
  - Linux
toc: true
math: true
type: about
---

cgroup和namespace类似，但是namespace会隔离不同进程组之间的资源，而cgroup是为了对进程组的资源进行限制和监控。

#### 组成

- `tasks`：在cgroup里面，一个tasks表示系统中一个进程组，在cgroup树中，一个tasks可以被认为是一个Node节点
- `subsystem`：一个资源调度器（Resource Controller）可以通过lssubsys -a命令查看当前内核支持哪些subsystem
- `hierarchy`(层级结构)： 一个hierarchy可以理解为一棵cgroup树，树的每个节点就是一个进程组，每棵树都会与零到多个subsystem关联。系统中可以有很多颗cgroup树，每棵树都和不同的subsystem关联，一个进程可以属于多颗树，即一个进程可以属于多个进程组，只是这些进程组和不同的subsystem关联。如果每个cgroup树不与subsystem关联，那么此时cgroup的作用也只是单纯的将进程分组而已

<img src="./Linux-cgroup/cgroups层级结构示意图.png" style="zoom: 25%;" />

##### 规程

参考：https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-relationships_between_subsystems_hierarchies_control_groups_and_tasks

###### Rule-1

<img src="./Linux-cgroup/RMG-rule1.png" style="zoom: 67%;" />

###### Rule-2

<img src="./Linux-cgroup/RMG-rule2.png" style="zoom:67%;" />

###### Rule-3

<img src="./Linux-cgroup/RMG-rule3.png" style="zoom:65%;" />

###### Rule-4

<img src="./Linux-cgroup/RMG-rule4.png" style="zoom:67%;" />

#### subsystem

- `blkio` ：限制进程的块设备 io
- `cpu` ：限制进程的 cpu 使用率
- `cpuacct`：统计 cgroups 中的进程的 cpu 使用报告
- `cpuset`：统计 cgroups 中的进程的 cpu 使用报告
- `devices` ：控制进程能够访问某些设备
- `freezer` ：挂起或者恢复 cgroups 中的进程
- `memory` ：限制进程的 memory 使用量
- `net_cls`：标记 cgroups 中进程的网络数据包，然后可以使用 tc 模块（traffic control）对数据包进行控制
- `ns` ：使不同 cgroups 下面的进程使用不同的 namespace
- `perf_event`：
- `net_prio`：

#### 挂载cgroup树

一般在sys/fs/cgroup下已经创建好各个与subsystem关联的cgroup树

##### 挂载

```
mkdir /sys/fs/cgroup/< subsystem name >
mount -t cgroup -o < subsystem name > <mount name> /sys/fs/cgroup/< subsystem name >
```

##### 查看

```
mount | grep cgroup
```

#### cgroup树目录

##### 根目录下主要文件

- cgroup.clone_children：该文件的内容为1时，当cgroup退出时（不再包含任何进程和子cgroup），将调用release_agent里面配置的命令。新cgroup被创建时将默认继承父cgroup的这项配置
- cgroup.event_control:用于eventfd的接口，可以在进程组状态发生改变时发送通知
- cgroup.procs ：当前cgroup中的所有进程ID
- notify_on_release：该文件的内容为1时，当cgroup退出时（不再包含任何进程和子cgroup），将调用release_agent里面配置的命令。新cgroup被创建时将默认继承父cgroup的这项配置
- tasks：当前cgroup中进程里面所有线程ID
- release_agent：里面包含了cgroup退出时将会执行的命令，系统调用该命令时会将相应cgroup的相对路径当作参数传进去。
- cgroup.sane_behavior 

子目录与根目录下文件基本一样，只不过子目录比根目录少了release_agent和cgroup.sane_behavior文件

##### 创建子memory group

```
cd /sys/fs/cgroup/memory
mkdir mycgroup
echo $pid > /sys/fs/cgroup/memory/mycgroup/cgroup.procs
echo 1M > /sys/fs/cgroup/memory/mycgroup/memory.max_usage_in_bytes
```



#### Reference

- https://www.kernel.org/doc/Documentation/cgroup-v2.txt
- https://www.kernel.org/doc/Documentation/
- https://tech.meituan.com/2015/03/31/cgroups.html
- https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-relationships_between_subsystems_hierarchies_control_groups_and_tasks

