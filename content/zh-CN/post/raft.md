---
title: "Raft"
date: "2023-05-10 18:23:11"
categories:
  - Etcd
toc: true
math: true
type: about
---

  etcd所使用的raft算法要点！  
                                  

### raft角色/状态
  * Leader: 领导者
  * Follower: 跟随者
  * Candidate: 候选人, 选举中的一个临时角色

### 选举
  1. 在一定时间内(ttl), follower没有收到来自leader的心跳(heartbear), 那么follower便会变为Candidate, 发送Request Vote到其他follower中, 进行leader选举和投票
  2. 其他follower收到Request Vote请求后, 便开始参与投票。投票完成之后，票数多的当选leader
  3. 选举出leader之后, leader会定期发送heartbeat消息给follower, 以此来告诉follower, leader还在线。

### 日志复制(Log Replication)
  1. leader收到client端请求, 每次操作请求都会产生日志条目(log entries)在节点(node)的日志中
  2. leader产生log entries之后, 此时还不能提交, 还要将该条日志复制发到follower中
  3. 待follower收到新增日志, leader收到大多数follower的确认回复后, leader确认该条消息已经更新提交，最后leader再通知follower更新提交

  注意: 如果client端是和follower进行交互, 那么follower会把消息重定向到leader上

### 日志一致性
  在 Raft 算法中，领导人leader是通过强制跟随者follower直接复制自己的日志来处理不一致问题的。这意味着在跟随者follower中的冲突的日志条目会被领导人leader的日志覆盖
#### 日志结构
  日志由有序序号标记的条目组成。每个条目都包含创建时的任期号，和一个状态机需要执行的指令。一个条目当可以安全地被应用到状态机中去的时候，就认为是可以提交了。

#### 日志匹配特性（Log Matching Property）
  * 如果在不同的日志中的两个条目拥有相同的索引和任期号，那么他们存储了相同的指令。
  * 如果在不同的日志中的两个条目拥有相同的索引和任期号，那么他们之前的所有日志条目也全部相同。

#### 过程
  译文如下：
  ```
  要使得跟随者的日志进入和自己一致的状态，领导人必须找到最后两者达成一致的地方，然后删除跟随者从那个点之后的所有日志条目，并发送自己在那个点之后的日志给跟随者。
  所有的这些操作都在进行附加日志 RPCs 的一致性检查时完成。领导人针对每一个跟随者维护了一个 `nextIndex`，这表示下一个需要发送给跟随者的日志条目的索引地址。
  当一个领导人刚获得权力的时候，他初始化所有的 `nextIndex` 值为自己的最后一条日志的 `index 加 1`。
  如果一个跟随者的日志和领导人不一致，那么在下一次的附加日志 RPC 时的 `一致性检查` 就会失败。
  在被跟随者拒绝之后，领导人就会减小 `nextIndex` 值并进行重试。最终 `nextIndex` 会在某个位置使得领导人和跟随者的日志达成一致。
  当这种情况发生，附加日志 RPC 就会成功，这时就会把跟随者冲突的日志条目全部删除并且加上领导人的日志。一旦附加日志 RPC 成功，那么跟随者的日志就会和领导人保持一致，并且在接下来的任期里一直继续保持。
  ```
  
  简单来说就是leader维护一个 `nextIndex` ，然后通过发送附加日志RPCs给follower，follower检查 `nextIndex` 和自己日志的index是否一致，不一致便拒绝该消息，leader得知follower拒绝之后
  就 `nextIndex - 1`, 继续该过程， 直到follwer接受，那么follwer就删除覆盖当前日志条目索引地址之后的日志，使得与leader的日志相同

#### 压缩

### Reference
  url: http://thesecretlivesofdata.com/raft/#home  
  url: https://github.com/MrVWY/raft-zh_cn/blob/master/raft-zh_cn.md