---
title: "BGP"
date: "2020-12-01 02:13:41"
categories:
  - 路由交换
tags:
  - 路由交换
toc: true
math: true
type: about
---

### BGP对等体之间的交互原则

- 从IBGP对等体获得的BGP路由，BGP设备只发布给它的EBGP对等体。
- 从EBGP对等体获得的BGP路由，BGP设备发布给它所有EBGP和IBGP对等体。
- 当存在多条到达同一目的地址的有效路由时，BGP设备只将最优路由发布给对等体。
  路由更新时，BGP设备只发送更新的BGP路由。
- 所有对等体发送的路由，BGP设备都会接收。

### BGP引入IGP路由

BGP协议本身不发现路由，因此需要将其他路由引入到BGP路由表，实现AS间的路由互通。当一个AS需要将路由发布给其他AS时，AS边缘路由器会在BGP路由表中引入IGP的路由。为了更好的规划网络，BGP在引入IGP的路由时，可以使用路由策略进行路由过滤和路由属性设置，也可以设置MED值指导EBGP对等体判断流量进入AS时选路。

- BGP引入路由时支持Import和Network两种方式：
  - Import方式是按协议类型，将RIP、OSPF、ISIS等协议的路由引入到BGP路由表中。为了保证引入的IGP路由的有效性，Import方式还可以引入静态路由和直连路由。
  - Network方式是逐条将IP路由表中已经存在的路由引入到BGP路由表中，比Import方式更精确。

### BGP Commuity

- Internet：表示所有路由，可以通过匹配Internet 属性来匹配所有的路由条目
- No_export：不通告给任何EBGP邻居，如果存在BGP联邦，则会在联邦内的EBGP邻居之间传递
- No_advertise：不通告给任何BGP邻居

### As_Path  Type

组成：Path Segment Type,  Path Segment Length,  Path Segment Value

- As_Set: 由一系列As号无序地组成，包含在Updata消息里
- As_Sequence: 由一系列As号顺序地组成，包含在Updata消息里
- As_Confed_Sequence: 在本地联盟内由一系列成员As号按顺序地组成，包含在Updata消息中，只能在本地联盟内传递
- As_Confed_Set: 在本地联盟内由一系列成员As无序地组成，包含在Updata消息中，同样只能在本地联盟内传递

### BGP Ring Protection

- As内部防环：通过iBGP水平分割实现。从iBGP邻居学到的路由不会更新给其他IBGP邻居。

  若无此机制，那么当三个路由器两两相连，并建立了iBGP邻居时，那么此时其中一个路由器发送的更新路由将在三个路由器之间无限循环

  要实现As内部每台路由器都可以学习到路由，需要建立iBGP邻居全互联（路由器数目很大的时候，那将导致网络中的路由器因为需要维护过多的BGP邻居关系导致性能下降）。可以通过路由反射器，eBGP联邦解决

- As间防环：通过 As path 防环，每经过一个As，会添加该As的As编号在As_Path字段的最前面,当从eBGP邻居得到一条路由时，会检查该路由的As_path字段有没有自身所在As，如果有则，丢弃，如果没有则继续

- RR防环

- SOO防环

Reference

- https://forum.huawei.com/enterprise/en/bgp-ring-protection-mechanisms/thread/481269-861

### BGP Route Reflector ( RR )

　在BGP的网络中，为保证IBGP对等体之间的连通性，需要在IBGP对等体之间建立全连接关系。假设在一个AS内部有n台路由器，那么应该建立的IBGP连接数就为n(n-1)/2.当IBGP对等体数目很多时，对网络资源和CPU资源的消耗都很大

​    在一个AS内，其中一台路由器作为路由反射器RR（Route Reflector），RR和Client组成i一个集群（Cluster），其它路由器作为客户机（Client）与路由反射器之间建立IBGP连接。路由反射器在客户机之间传递（反射）路由信息，而客户机之间不需要建立BGP连接

​    既不是反射器也不是客户机的BGP路由器被称为非客户机（Non-Client）。非客户机与路由反射器之间，以及所有的非客户机之间仍然必须建立全连接关系

- 总结：

  - Client只需维护与RR之间的IBGP会话

  - Non-Client与Non-Client之间需要建立IBGP全互连
  - RR与Non-Client之间需要建立IBGP全互连
  - RR与RR之间需要建立IBGP的全互连


- rule

  -   从非客户机IBGP对等体学到的路由，发布给此RR的所有客户机

  - 从EBGP对等体学到的路由，发布给所有的非客户机和客户机
  - 从客户机学到的路由，发布给此RR的所有非客户机和客户机（发起此路由的客户机除外）
  - 从EBGP对等体学到的路由，发布给所有的非客户机和客户机


- Router Reflector Cluster
  - 当一个AS内存在多台RR为Client提供冗余时，RR间的路由更新很有可能会形成环路，为防止该现象，引入簇（Cluster）的概念

  -  通过4字节的Cluster_ID来标识Cluster，通常会使用Loopback地址作为Cluster_ID
  - 一个Cluster里可以包括一个或多个RR；一个Client可以同时属于多个Cluster
  - 通常，一个客户的簇只拥有一个RR，并由RR的BGP Router-id去标识该簇。有时，为了防止单点失效，在单一簇里引入多个RR

- Ring Protection

  - Originator_ID
    - Originator_ID属性用于防止在反射器和客户机/非客户机之间产生环路
    - Originator_ID属性长4字节，可选非过渡属性，属性类型为9 ，是由路由反射器（RR）产生的，携带了本地AS内部路由发起者的Router ID
    - 当一条路由第一次被RR反射的时候，RR将Originator_ID属性加入到这条路由，标识这条路由的始发路由器。如果一条路由中已经存在了Originator_ID属性，则RR将不会创建新的Originator_ID
    - 当其它BGP Speaker接收到这条路由的时候，将比较收到的Originator_ID和本地的Router ID，如果两个ID相同，BGP Speaker会忽略掉这条路由，不做处理

  - Cluster_List 
    - Originator_ID属性用于防止在RR之间产生环路
    - Cluster_List是可选非过渡属性，属性类型编码为10
    - Cluster_List由一系列的Cluster_ID组成，描述了一条路由所经过的反射器路径，这和描述路由经过的As路径的AS_Path属性有相似之处。Cluster_List由路由反射器产生
    - Cluster_List只在AS内部传播，从EBGP对等体收到的含有Cluster_List的路由将被丢弃
      当RR在它的客户机之间或客户机与非客户机之间反射路由时，RR会把本地Cluster_ID添加到Cluster_List的前面。如果Cluster_List为空，RR就创建一个
    - 当RR接收到一条更新路由时，RR会检查Cluster_List。如果Cluster_List中已经有本地Cluster_ID，丢弃该路由不需要再反射；如果没有本地Cluster_ID，将其加入Cluster_List，然后反射该更新路由
    - Cluster_List只被RR用来检测路由环路，不是RR的客户机和非客户机不会检测该属性

### BGP Confederation

将一个大的As分割成多个小型的As，让As内部拥有足够数量的eBGP邻居关系来解决路由限制问题。

对于外部邻居来说(联盟外的的对等体)，成员AS拓扑是不可见的。也就是说，在发向eBGP邻居的更新消息中，已经剥去了联盟内被修改的As_PATH。从其他的自治系统来看，联盟就像单个As一样。

- Ring Protection

  Confederation内采用As_Confed防止子AS间路由环路


- Confederation内As_Path属性变化

  ```
  //示意图
              |-------->  As 200 <----------------|            
  As(100) --> As((1000),100) --> As((1000,1001),100) --> As(200,100)
  ```

  - Confederation内eBGP
    - 子As号被添加到As_Path中的As_Confed_Sequence前面

  - Confederation内iBGP
    - 不做任何改变
  - 外部eBGP
    - As号从As_Path中清除，而大As号被添加到As_Path前面

### Routing Priority

### BGP Routing Switching Principles

从上到下

- Weight：优先选择 highest weight 的路由（ cisco私有属性 ）
- Local Preference：优先选择 highest local preference 的路由
- Originate：优先选择自己本地的路由
- As path length：优先选择 shortest As path length 的路由
- Origin code：优先选择 lowest origin code 的路由 ( IGP < EGP < incomplete)
- MED：优先选择 lowest MED 的路由
- eBGP path over iBGP path：eBGP的路由优于iBGP的路由
- Shortest IGP path to BGP next hop：
- Oldest Path：从更老的eBGP邻居学过来的路由，因为eBGP邻居建立时间越久，说明越稳定
- Router ID：优先选择lowest BGP neighbor router ID的路由
- Neighbor IP address：优先选择lowest neighbor IP address的路由

- Reference
  - https://networklessons.com/bgp/bgp-attributes-and-path-selection

### BGP Tables

- BGP Neighbor Table：包含有关BGP邻居信息的表
- BGP Table (BGP RIB)：包含从network layer reachability information (NLRI)学习到的路由和来自所以邻居的所以路由
- BGP Routing Table：包含由BGP Table的路由中选择最优路由

### BGP  Message Format Field

- Marker: Included for compatibility, must be set to all ones.
- Length: Total length of the message in [octets](https://en.wikipedia.org/wiki/Octet_(computing)), including the header.
- Type：Type of BGP message. The following values are defined:
  - Open (1)：发送一些参数以协商和建立连接，两个邻居都会发送Open报文，一个先发Open，另一个收到后，发送一个Open+Keeplive报文。Open报文里携带了version、BGP AS号、Holdtime、BGP identifier(router-id)
  - Update (2)：用于对等体之间交换路由信息
  - Notification (3)：只要检测到error，便发送并关闭BGP连接
  - KeepAlive (4)：周期性（Holdtime）发送以确认邻居是否存活 （extend：Error Code, Error Subcode）
  - Route-Refresh (5)：用来要求对等体重新发送指定地址族的路由信息

#### Message Header Format

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |                                                               |
      +                                                               +
      |                                                               |
      +                                                               +
      |                           Marker                              |
      +                                                               +
      |                                                               |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |          Length               |      Type     |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- BGP报文由BGP报文头和具体报文内容两部分组成

- BGP报文头包括三的部分，总长19字节。

  - Marker：占16字节，用于检查BGP对等体的同步信息是否完整，以及用于BGP验证的计算。不使用验证时所有比特均为1（十六进制则全“FF”）。

  - Length：占2个字节（无符号位），BGP消息总长度（包括报文头在内），以字节为单位。长度范围是19～4096。

  - Type：占1个字节（无符号位），BGP消息的类型。Type有5个可选值，表示BGP报文头后面所接的5类报文（其中，前四种消息是在RFC4271中定义的，而Type5的消息则是在RFC2918中定义的）

    	TYPE值  报文类型
    		1	OPEN
    		2	UPDATE
    		3	NOTIFICATION
    		4	KEEPALIVE
    		5	REFRESH

#### Open Message Format

```
       0                   1                   2                   3
       0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+
       |    Version    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |     My Autonomous System      |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |           Hold Time           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         BGP Identifier                        |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       | Opt Parm Len  |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                                                               |
       |             Optional Parameters (variable变长)                |
       |                                                               |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- version：表示协议的版本号，现在BGP的版本号为4。
- My autonomous System:发送者自己的AS域号
- Hold Time:发送者自己设定的hold time值（单位：秒），用于协商BGP对等体间保持建立连接关系，发送KEEPALIVE或UPDATE等报文的时间间隔。BGP的状态机必须在收到对等体的OPEN报文后，对发出的OPEN报文和收到的OPEN报文两者的hold time时间作比较，选择较小的时间作为协商结果。Hold Time的值可为零（不发KEEPALIVE报文）或大于等于3，我们系统的默认为180。
- BGP Identifier：发送者的router id。
- Opt Parm Len：表示Optional Parameters（可选参数）的长度。如果此值为0，表示没有可选参数。
- Optional Paramters：此值为BGP可选参数列表，每一个可选参数是一个TLV格式的单元(RFC3392)。

#### Update Message Format

```
      +-----------------------------------------------------+
      |   Withdrawn Routes Length (2 octets)                |
      +-----------------------------------------------------+
      |   Withdrawn Routes (variable变长)                   |
      +-----------------------------------------------------+
      |   Total Path Attribute Length (2 octets)            |
      +-----------------------------------------------------+
      |   Path Attributes (variable变长)                    |
      +-----------------------------------------------------+
      |Network Layer Reachability Information (variable变长)|
      +-----------------------------------------------------+
```

- Unfeasible Routes Length:标明Withdrawn Routes部分的长度。其值为零时，表示没有撤销的路由。
- Withdrawn Routes:包含要撤销的路由列表，列表中的每个单元包含1字节的Length域和可变长度的Prefix域。
- Total Path Attribute Length：标明Path Attributes部分的长度。其值为零时，表示没有路由及其路由属性要通告。
- Path_Attributes：含要更新的路由属性列表，按其类型号从小到大的顺序排序，填写更新的路由的所有属性。每一个属性单元包括属性类型，属性长度，属性值三部分。
- Network Layer Reachability Information（NLRI）：含要更新的地址前缀列表，每一个地址前缀单元由一个LV二元组（prefix length, the prefix of the reachable route）组成，其编码填写方法与Withdrawn Routes的填写方法相同。

```
--------------------路由属性的类型号列表-------------------------------
属性类型	属性值
1：Origin	IGP
			EGP
			Incomplete
2：As_Path	AS_SET
			AS_SEQUENCE
			AS_CONFED_SET
			AS_CONFED_SEQUENCE
3：Next_Hop	下一跳的IP地址
4：Multi_Exit_Disc	MED用于判断流量进入AS时的最佳路由
5：Local_Pref	Local_Pref用于判断流量离开AS时的最佳路由
6：Atomic_Aggregate	BGP Speaker选择聚合后的路由，而非具体的路由
7：Aggregator	发起聚合的路由器ID和AS号
8：Community	团体属性
9：Originator_ID	反射路由发起者的Router ID
10：Cluster_List	反射路由经过的反射器列表
14：MP_REACH_NLRI	多协议可达NLRI
15：MP_UNREACH_NLRI	多协议不可达NLRI
16：Extended Communtities	扩展团体属性
```

#### Notification Message Format

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      | Error code    | Error subcode |   Data (variable变长)         |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- Error code：占1个字节（无符号位），定义错误的类型，非特定的错误类型用零表示。
- Error subcode：占1个字节（无符号位），指定错误细节编号，非特定的错误细节编号用零表示。
- Data：指定错误数据内容。

#### Keepalive Message Format

Keepalive 报文只有BGP报文头，没有具体内容，故其报文长度应固定为19个字节。

#### Refresh Message Format

Refresh 报文用于动态的请求BGP路由发布者重新发布Update报文，进行路由更新。

### BGP Connection State

- Idle State

  在空闲状态，BGP在等待一个启动事件，启动事件出现以后，BGP初始化资源，复位连接重试计时器（Connect-Retry），发起一条TCP连接，同时转入Connect 状态

  在BGP FSM中，发生任何错误，BGP session都将中止会话并返回该状态

- Connect State

  - 如果TCP Negotiation (TCP三次握手)成功，BGP将 connect Retry 清零，完成初始化并发送一个 open 消息包给邻居并把自己的状态置为 Open send 状态 
  - 如果失败，BGP会继续监听邻居发出的连接，重置 Connect Retry 并将自己状态转移到 Active 状态
  - 如果 Connect Retry 时间超时，将重新开始，再次试图与邻居建立TCP连接，BGP 保持 Connect 状态，出现其他事件转入 Idle 状态

- Active State

  - 如果 TCP 连接成功，BGP 将 Connect Retry 清零，完成初始化，给邻居发送Open 消息并将状态置为 Open，hold 时间置为4mins 
  - 如果在 Active 状态，Connect Retry 超时回 Connect 状态并重置 Connect Retry 计时器，如果再次连接失败，会导致重回 Idle 状态

- OpenSent State

  该状态是在三次握手成功后，发送了 open 报文后，等待对端回应 open 报文时的状态

  - 收到open消息，如果发现有差错，将给邻居发送一个 notification 消息并将状态置为 Idle 
  - 如果收到 open 包消息校验正确，将发送 keeplive 包给邻居，并建立IBGP或者EBGP状态置为 Open confire state ，否则
  - 如果收到TCP断开消息则断开BGP连接重置 Connect Retry ，状态置为 Active 

- OpenConfirm State

  该状态，BGP等待对端的 keepalive 报文，收到后，转入 Established 状态

  - 如果收到一个 keeplive 消息包，会将状态置为 Establish 

  - 如果收到 notification 消息包，会将状态置为 Idle 并断开 TCP 连接

- Established State

  此状态下 BGP 对等体间的连接已经完全建立，此时可以相互交换路由信息，若收到 notification，则会将状态置为 Idle 中断连接

- 示意图：

![](/images/Review-the-BGP-protocol/202103051314520.png)

### BGP Synchronization

```
//拓补图
AS100(RTA) --------- AS200(RTB-RTC-RTD) ---------- AS300(RTE)
```

当AS100中的RTA发送关于1.1.1.1的路由update包，RTB和RTD运行着IBGP，理想情况下RTD在update包中学习到如何到达1.1.1.1的网络信息，接着RTD传播update包给RTE

假如RTB没有将1.1.1.1的路由信息重发布到IGP中，那么RTC便无法得知如何去往1.1.1.1这段网络，那么RTD和RTE如果向1.1.1.1网段发送数据包，便会在RTC这被丢弃

BGP同步规则：BGP路由器不应该使用或向EBGP邻居通告从IBGP邻居那里学习到的BGP路由信息，除非该路由是本地的或者该路由存在于IGP数据库，即该路由也能从IGP学习到

```
The BGP synchronization rule states that if an AS provides transit service to another AS, BGP should not advertise a route until all of the routers within the AS have learned about the route via an IGP.
```

### BGP Router Server

IX/IXP的理念就是帮助不同运营商之间为连通各自网络而建立的集中交换平台，而多个As通过IX/IXP互联则需要在多个As边界路由器之间建立Full的eBGP邻居关系，很大程度上浪费网络资源和本身As边界路由器的Cpu和memory

![](/images/Review-the-BGP-protocol/1615539343(1).jpg)
而引入了BGP Router Server之间，每个As边界路由器只需要与其建立eBGP关系即可

![](/images/Review-the-BGP-protocol/1615539586(1).jpg)
个人感觉BGP Router Server 有点类似与BGP Router Reflector 

- Reference
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-3s/irg-xe-3s-book/irg-route-server.pdf
  - [Internet Exchange Point ( IX / IXP )](https://baike.baidu.com/item/%E4%BA%92%E8%81%94%E7%BD%91%E4%BA%A4%E6%8D%A2%E4%B8%AD%E5%BF%83/2948800?fr=aladdin)
  - [IX history](https://www.weibo.com/p/23041815dce34dd0102w5u5?mod=zwenzhang)

### BGP Flowspec 

BGP Flow Spec可以在BGP路由黑洞的基础上，对流量进行更加细致的分类、限速、过滤和重定向等操作，而不是像路由黑洞那样"一棒打死"

- Reference
  - https://github.com/osrg/gobgp/blob/master/docs/sources/flowspec.md
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/iproute_bgp/configuration/xe-3s/irg-xe-3s-book/C3PL-BGP-Flowspec-Client.pdf

### BMP

BMP能够对网络中的设备的BGP运行状态进行实时监控。BMP会话是单向的，即设备向监控服务器上报消息但忽略监控服务器发送的任何消息

- Common Header：

  ```
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+
       |    Version    |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                        Message Length                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |   Msg. Type   |
       +---------------+
      
  ```

- Message Type

  - Type = 0: Route Monitoring（RM）：路由监控消息，向监控服务器发送从对等体收到的所有路由，并随时向监控服务器上报路由的新增或撤销
  - Type = 1: Statistics Report（SR）：向监控服务器上报设备运行状态的统计信息
  - Type = 2: Peer Down Notification：向监控服务器上报与对等体BGP连接的中断
  - Type = 3: Peer Up Notification：向监控服务器上报与对等体BGP连接的建立
  - Type = 4: Initiation Message：初始化消息，向监控服务器通告厂商信息、版本号等
  - Type = 5: Termination Message：结束消息，向监控服务器通告关闭BMP会话的原因
  - Type = 6: Route Mirroring Message

- Reference
  
  - [BGP Monitoring Protocol (BMP)](https://tools.ietf.org/html/rfc7854)

### BFD

BFD（Bidirectional Forwarding Detection，双向转发检测）是一种快速故障检测机制，常用于链路的连通性检测，可以为各种上层协议（静态路由、Track等）快速检测设备之间的路径故障

### BGP安全性

- BGP GTSM（Generalized TTL Security Mechanism）

  BGP GTSM检测IP报文头中的TTL（time-to-live）值是否在一个预先设置好的特定范围内，并对不符合TTL值范围的报文进行允许通过或丢弃的操作

- BGP 认证

  MD5认证、Keychain认证

- BGP RPKI

  RPKI（Resource Public Key Infrastructure）通过验证BGP路由起源是否正确来保证BGP的安全性。

  RPKI主要用来解决路由传递过程中攻击者通过发布更精确的路由导致数据传输过程中流量被窃取的问题。例如，某运营商向用户发布了目的地址为10.10.0.0/16的路由，路由攻击者发布了目的地址为10.10.153.0/16的路由，由于攻击者发布的路由更详细，从用户端发往10.10.153.0/16的路由将被攻击者窃取。

  为了解决上述问题，路由器与RPKI服务器建立连接，将路由起源的相关数据ROA（Route Origination Authorization）保存到本地。通过验证从邻居收到的BGP路由是否合法来控制选路结果，从而确保域内的主机能够安全地访问外部服务。

### Route Flapping

路由不稳定的主要表现形式是路由振荡（Route Flapping），即路由表中的某条路由反复消失和重现

发生路由振荡时，路由器就会向邻居发布路由更新，收到更新报文的路由器需要重新计算路由并修改路由表。所以频繁的路由振荡会消耗大量的带宽资源和CPU资源，严重时会影响到网络的正常工作

路由衰减（Route Dampening）用来解决路由不稳定的问题。多数情况下，BGP协议都应用于复杂的网络环境中，路由变化十分频繁。为了防止持续的路由振荡带来的不利影响，BGP使用路由衰减来抑制不稳定的路由

BGP衰减使用惩罚值（Penalty Value）来衡量一条路由的稳定性，惩罚值越高则说明路由越不稳定。路由每发生一次振荡，BGP便会给此路由增加一定的惩罚值（路由从激活状态变为未激活状态，惩罚值增加1000，路由在激活状态，收到新的路由更新，惩罚值加500）。当惩罚值超过抑制阈值（Suppress Value）时，此路由被抑制，不加入到路由表中，也不再向其他BGP对等体发布更新报文

被抑制的路由每经过一段时间，惩罚值便会减少一半，这个时间称为半衰期（Half-life）。当惩罚值降到再使用阈值（Reuse Value）时，此路由变为可用并被加入到路由表中，同时向其他BGP对等体发布更新报文。上文提到的惩罚值、抑制阈值和半衰期都可以手动配置

路由衰减只适用于EBGP路由。对于从IBGP收来的路由不能进行衰减，因为IBGP路由经常含有本AS的路由，内部网络路由要求转发表尽可能一致，IGP快速收敛就是为了达到信息同步，转发一致。如果衰减对IBGP路由起作用，不同路由器的衰减参数不一致时，会导致转发表不一致

### BGP Auto FRR

BGP Auto FRR（Auto Fast ReRoute）是一种链路故障保护措施，应用于有主备链路的网络拓扑结构中。使能BGP Auto FRR，可以使BGP的两个邻居切换或者两个下一跳切换达到亚秒级的收敛速度

BGP Auto FRR对于从不同对等体学到的相同前缀的路由，利用最优路由作为主链路进行转发，并自动将次优路由作为备份链路。当主链路出现故障的时候，系统快速响应BGP路由不可达的通知，并将转发路径切换到备份链路上

### BGP Aggregate

将多个特定路由聚合为一个路由，可能会有路径属性丢失。	可分为手动聚合和自动聚合
为了避免路由聚合可能引起的路由环路，BGP设计了AS_Set属性。AS_Set属性是一种无序的AS_Path属性，标明聚合路由所经过的AS号。当聚合路由重新进入AS_Set属性中列出的任何一个AS时，BGP将会检测到自己的AS号在聚合路由的AS_Set属性中，于是会丢弃该聚合路由，从而避免了路由环路的形成。

### BGP GR and NSR

BGP的平滑重启GR（Graceful Restart）和不间断路由NSR（Non-Stop Routing）作为高可靠性的解决方案，其根本目的都是为了保证用户业务在设备故障的时候不受影响或者影响最小

GR机制的核心在于：当某设备进行协议重启时，能够通知其周边设备在一定时间内将到该设备的邻居关系和路由保持稳定。在协议重启完毕后，周边设备协助其进行信息（包括支持GR的路由/MPLS相关协议所维护的各种拓扑、路由和会话信息）同步，在尽量短的时间内使该设备恢复到重启前的状态。在整个协议重启过程中不会产生路由振荡，报文转发路径也没有任何改变，整个系统可以不间断地转发数据。这个过程即称为平滑重启。

GR基本概念
配置了GR功能的设备称为“具备GR能力”的设备。具备GR能力的设备在协议重启时，能实现平滑重启，保证转发业务不中断；而不具备GR能力的设备在协议重启时，则只能遵循普通的重启过程。GR中涉及到的基本概念如下：

1. GR Restarter
GR重启路由器，指由管理员或故障触发而协议重启的设备，它必须具备GR能力。
2. GR Helper
即GR Restarter的邻居，能协助重启的GR Restarter保持路由关系的稳定，它也必须具备GR能力。
3. GR Session
GR会话，是GR Restarter和GR Helper之间的协商过程。包括协议重启通告，协议重启过程中的信息交互等。通过该会话，GR Restarter和GR Helper可以掌握彼此的GR能力。
4. GR Time
GR时间，是GR Restarter和GR Helper协商建立一个会话所用的时间。当某GR路由器发现邻居路由器处于down状态时，将在该时间内仍保留其发出的拓扑或路由信息。

BGP GR的过程是：
利用BGP的能力协商机制，GR Restarter和GR Helper了解彼此的GR能力，建立有GR能力的会话。
当GR Helper检查到GR Restarter重启或者主备倒换后，不删除和GR Restarter相关的路由和转发表项，也不通知其他邻居，而是等待重建BGP连接。
GR Restarter在GR Time超时前与重启前的所有GR Helper新建立好邻居关系。

BGP NSR（Nonstop Routing，不间断路由）是一种通过在BGP协议主备进程之间备份必要的协议状态和数据（如BGP邻居信息和路由信息），使得BGP协议的主进程中断时，备份进程能够无缝地接管主进程的工作，从而确保对等体感知不到BGP协议中断，保持BGP路由，并保证转发不会中断的技术。

### BGP 定时器

- Advertisement Interval：发布路由通告的间隔。在这个间隔内的事件会被缓存，然后时间到了一起发送
- Keepalive and Hold Timers：每个节点会向它的 peer 发送心跳消息。如果一段时间内（称为 hold time）没收到 peer 的心跳，就会清除所有从这个 peer 收到的消息
- Connect Timer：节点和 peer 建立连接失败后，再次尝试建立连接之前需要等待的时长

### BGP Tracking

为了实现BGP快速收敛，可以通过配置BFD来探测邻居状态变化，但BFD需要全网部署，扩展性较差。在无法部署BFD检测邻居状态时，可以本地配置BGP Peer Tracking功能，快速感知链路不可达或者邻居不可达，实现网络的快速收敛。
通过部署BGP Tracking功能，调整从发现邻居不可达到中断连接的时间间隔，可以抑制路由震荡引发的BGP邻居关系震荡，提高BGP网络的稳定性。

### BGP Redistributes Routes

一种协议收到的路由以另一种协议再发送出去，称为路由再分发（redistributing routes）

### MP-BGP

### MP-BGP EVPN

#### BGP EVPN路由类型

传统的BGP-4使用Update报文在对等体之间交换路由信息。一条Update报文可以通告一类具有相同路径属性的可达路由，这些路由放在NLRI（Network Layer Reachable Information，网络层可达信息）字段中。

因为BGP-4只能管理IPv4单播路由信息，为了提供对多种网络层协议的支持（例如IPv6、组播），发展出了MP-BGP（MultiProtocol BGP）。MP-BGP在BGP-4基础上对NLRI作了新扩展。玄机就在于新扩展的NLRI上，扩展之后的NLRI增加了地址族的描述，可以用来区分不同的网络层协议，例如IPv6单播地址族、VPN实例地址族等。

类似的，EVPN在L2VPN地址族下定义了新的子地址族——EVPN地址族，并新增了一种NLRI，即EVPN NLRI。EVPN NLRI定义了以下几种BGP EVPN路由类型，通过在EVPN对等体之间发布这些路由，就可以实现VXLAN隧道的自动建立、主机地址的学习。

- Type2路由——MAC/IP路由：用来通告主机MAC地址、主机ARP和主机路由信息。
- Type3路由——Inclusive Multicast路由：用于VTEP的自动发现和VXLAN隧道的动态建立。
- Type5路由——IP前缀路由：用于通告引入的外部路由，也可以通告主机路由信息。

EVPN路由在发布时，会携带RD（Route Distinguisher，路由标识符）和VPN Target（也称为Route Target）。RD用来区分不同的VXLAN EVPN路由。VPN Target是一种BGP扩展团体属性，用于控制EVPN路由的发布与接收。也就是说，VPN Target定义了本端的EVPN路由可以被哪些对端所接收，以及本端是否接收对端发来的EVPN路由。

VPN Target属性分为两类：

- Export Target（ERT）：本端发送EVPN路由时，将消息中携带的VPN Target属性设置为Export Target。
- Import Target（IRT）：本端在接收到对端的EVPN路由时，将消息中携带的Export Target与本端的Import Target进行比较，只有两者相等时才接收该路由，否则丢弃该路由。

#### Type 2 类型路由

![](/images/Review-the-BGP-protocol/Type2类型路由中NLRI格式.png)

#### Type 3 类型路由
![](/images/Review-the-BGP-protocol/Type3类型路由中NLRI格式.png)

#### Type 5 类型路由
![](/images/Review-the-BGP-protocol/Type5类型路由中NLRI格式.png)