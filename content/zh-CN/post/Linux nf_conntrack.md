---
title: "linux nf_conntrack"
date: "2021-04-13 15:34:10"
categories:
  - Linux
tags:
  - Linux
toc: true
math: true
type: about
---

nf_conntrack是一个内核模块，其用于跟踪一个连接的状态。

#### 为什么需要跟踪连接的状态？

nf_conntrack允许内核”审查”通过此处的所有网络数据包，并能识别出此数据包属于哪条连接。因此，连接跟踪机制使内核能够跟踪并记录通过此处的所有网络连接及其状态。

结合Iptables使用，这条 INPUT 规则用于放行 80 端口上的状态为 NEW 的连接上的包。

```
iptables -A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
```

#### 状态

- `New`：匹配连接的第一个包（分组对应的连接在两个方向上都没有进行过分组传输）。意思就是，iptables从连接跟踪表中查到此包是某连接的第一个包。判断此包是某连接的第一个包是依据conntrack当前”只看到一个方向数据包”([UNREPLIED])，不关联特定协议，因此NEW并不单指tcp连接的SYN包

- `Established`：匹配连接的响应包及后续的包。意思是，iptables从连接跟踪表中查到此包是属于一个已经收到响应的连接(即没有[UNREPLIED]字段)。因此在iptables状态中，只要发送并接到响应，连接就认为是Established的了。这个特点使iptables可以控制由谁发起的连接才可以通过，比如A与B通信，A发给B数据包属于NEW状态，B回复给A的数据包就变为Established状态。ICMP的错误和重定向等信息包也被看作是Established，只要它们是我们所发出的信息的应答

- `Related`：匹配那些属于Related连接的包，这句话说了跟没说一样。Related状态有点复杂，当一个连接与另一个已经是Established的连接有关时，这个连接就被认为是Related。这意味着，一个连接要想成为Related，必须首先有一个已经是Established的连接存在。这个Established连接再产生一个主连接之外的新连接，这个新连接就是Related状态了，当然首先Conntrack模块要能”读懂”它是Related。拿ftp来说，FTP数据传输连接就是Related与先前已建立的FTP控制连接，还有通过IRC的DCC连接。有了Related这个状态，ICMP错误消息、FTP传输、DCC等才能穿过防火墙正常工作。有些依赖此机制的TCP协议和UDP协议非常复杂，他们的连接被封装在其它的TCP或UDP包的数据部分(可以了解下overlay/vxlan/gre)，这使得Conntrack需要借助其它辅助模块才能正确”读懂”这些复杂数据包，比如nf_conntrack_ftp这个辅助模块

- `Invalid`：匹配那些无法识别或没有任何状态的数据包。这可能是由于系统内存不足或收到不属于任何已知连接的ICMP错误消息。一般情况下我们应该DROP此类状态的包

- `Untracked`：状态比较简单，它匹配那些带有NOTRACK标签的数据包。需要注意的一点是，如果你在raw表中对某些数据包设置有NOTRACK标签，那上面的4种状态将无法匹配这样的数据包，因此你需要单独考虑NOTRACK包的放行规则

#### 如何存储连接状态

nf_conntrack 主要利用hash来存储其跟踪的连接

- 源码参考连接

  https://code.woboq.org/linux/linux/net/netfilter/nf_conntrack_core.c.html

##### nf_conntrack_tuple

- 主要存储src和dst
- 通过下面源码，可以看出其主要支持tcp、udp、icmp、dccp、sctp、gre (具体看最新源码)

```
struct nf_conntrack_tuple {
                                                                         {
									 { union nf_inet_addr u3; -------| __be32  ip;
									 |							   { ..........
	struct nf_conntrack_man src;  ------|                                     { struct { __be16 port; } tcp
									 | union nf_conntrack_man_proto u;-----| struct { __be16 port; } udp
									 |                                     | struct { __be16 port; } icmp
									 |                                     | struct { __be16 port; } dccp
									 |                                     | struct { __be16 port; } sctp
									 |                                     { struct { __be16 port; } gre
									 { u_int16_t l3num; /* Layer 3 protocol */ 
	/* These are the parts of the tuple which are fixed. */
	struct {
		union nf_inet_addr u3;
		union {
			/* Add other protocols here. */
			__be16 all;
			struct {
				__be16 port;
			} tcp;
			struct {
				__be16 port;
			} udp;
			struct {
				u_int8_t type, code;
			} icmp;
			struct {
				__be16 port;
			} dccp;
			struct {
				__be16 port;
			} sctp;
			struct {
				__be16 key;
			} gre;
		} u;
		/* The protocol. */
		u_int8_t protonum;
		/* The direction (for tuplehash) */
		u_int8_t dir;
	} dst;
};
```

##### hash_conntrack_raw函数

利用 Tuple 的src、dst字段来计算哈希

```
static u32 hash_conntrack_raw(const struct nf_conntrack_tuple *tuple, const struct net *net)
{
	unsigned int n;
	u32 seed;
	get_random_once(&nf_conntrack_hash_rnd, sizeof(nf_conntrack_hash_rnd));
	/* The direction must be ignored, so we hash everything up to the
	 * destination ports (which is a multiple of 4) and treat the last
	 * three bytes manually.
	 */
	seed = nf_conntrack_hash_rnd ^ net_hash_mix(net);
	n = ( sizeof( tuple->src ) + sizeof( tuple->dst.u3 ) ) / sizeof(u32);
	return jhash2((u32 *)tuple, n, seed ^ (((__force __u16)tuple->dst.u.all << 16) |
		      tuple->dst.protonum));
}
```

因此当需要查找数据包是属于哪个连接，可以通过计算hash值去hash表寻找是否存有。

#### 常用Cmd

```
查看nf_conntrack表当前连接数    
cat /proc/sys/net/netfilter/nf_conntrack_count       

查看nf_conntrack表最大连接数  
cat /proc/sys/net/netfilter/nf_conntrack_max    

通过dmesg可以查看nf_conntrack的状况：
//小心nf_conntranck : table full, dropping packet
dmesg |grep nf_conntrack

查看存储conntrack条目的哈希表大小,此为只读文件
cat /proc/sys/net/netfilter/nf_conntrack_buckets

查看nf_conntrack的TCP连接记录时间
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established

通过内核参数查看命令，查看所有参数配置
sysctl -a | grep nf_conntrac
```

#### Iptables使用

```
iptables -A INPUT -p tcp -m state --state (NEW,ESTABLISHED,RELATED,INVALID) -m tcp --dport 80 -j ACCEPT
```

#### 相关参数

详见：https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt

- nf_conntrack_max：连接跟踪表的大小，
  根据wiki上的计算公式 CONNTRACK_MAX = RAMSIZE (in bytes) / 16384 / (x / 32)，并满足nf_conntrack_max=4*nf_conntrack_buckets，默认262144
- nf_conntrack_buckets：哈希表的大小，(nf_conntrack_max/nf_conntrack_buckets就是每条哈希记录链表的长度)，默认65536
- nf_conntrack_tcp_timeout_established：tcp会话的超时时间，默认是432000 (5天)

#### Tool

文档：http://conntrack-tools.netfilter.org/manual.html

```
yum install conntrack-tools
```

#### Reference

- https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE
- https://wiki.khnet.info/index.php/Conntrack_tuning
- https://wiki.aalto.fi/download/attachments/69901948/netfilter-paper.pdf