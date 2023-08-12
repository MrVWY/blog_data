---
title: "OSPF"
date: "2021-01-02 14:37:12"
categories:
  - 路由交换
tags:
  - 路由交换
toc: true
math: true
type: about
---

Ospf作为链路状态路由协议，路由器之间交互的是LSA（链路状态通告），路由器将网络中泛洪的LSA搜集到自己的LSDB（链路状态数据库）中，这有助于Ospf理解整张网络拓扑，并在此基础上通过SPF最短路径算法计算出以自己为根的、到达网络各个角落的、无环的树，最终，路由器将计算出来的路由装载进路由表中。

### Router ID

在DR和BDR选举的过程中，如果两台路由器的DR优先级相等，需要进一步比较两台路由器的Router ID，Router ID大的路由器将被选为DR或BDR。

Router ID是用于在自治系统中唯一标识一台运行OSPF的路由器的32位整数。每个运行OSPF的路由器都有一个Router ID。Router ID的格式和IP地址的格式是一样的。

### Cost

OSPF使用cost“开销”作为路由度量值。每一个激活OSPF的接口都有一个cost值，一条OSPF路由的cost由该路由从路由的起源一路到达本地的所有入接口cost值的总和。

### Ospf Tables

- 邻居表（Peer Table）：主要记录形成邻居关系路由器
- 链路状态数据库（LSDB）：Ospf用LSA（link state Advertisement 链路状态通告）来描述网络拓扑信息，然后Ospf路由器用链路状态数据库来存储网络的这些LSA。OSPF将自己产生的以及邻居通告的LSA搜集并存储在链路状态数据库LSDB中
- OSPF路由表（Routing Table）：对链路状态数据库进行SPF（Dijkstra）计算，而得出的OSPF路由表

### 五种报文

| 报文类型                              | 报文作用                                                     |
| ------------------------------------- | ------------------------------------------------------------ |
| Hello报文                             | 周期性发送，用来发现和维持OSPF邻居关系                       |
| DD报文（DataBase Description packet） | 描述本地LSDB（Link State Database）的摘要信息，用于两台设备进行数据库同步 |
| LSR报文（Link State Request packet）  | 用于向对方请求所需的LSA。设备只有在OSPF邻居双方成功交换DD报文后才会向对方发出LSR报文 |
| LSU报文（Link State Update packet）   | 用于向对方发送其所需要的LSA                                  |
| LSAck                                 | 用来对收到的LSA进行确认                                      |

### Ospf状态机

#### OSPF接口状态机

- Down：接口的初始状态。表明此时接口不可用，不能用于收发流量。
- Loopback：设备到网络的接口处于环回状态。环回接口不能用于正常的数据传输，但可以通过Router-LSA进行通告。因此，进行连通性测试时能够发现到达这个接口的路径。
- Waiting：设备正在判定网络上的DR和BDR。在设备参与DR和BDR选举前，接口上会启动Waiting定时器。在这个定时器超时前，设备发送的Hello报文不包含DR和BDR信息，设备不能被选举为DR或BDR。这样可以避免不必要地改变链路中已存在的DR和BDR。仅NMBA网络、广播网络有此状态。
- P-2-P：接口连接到物理点对点网络或者是虚拟链路，这个时候设备会与链路连接的另一端设备建立邻接关系。仅P2P、P2MP网络有此状态。
- DROther：设备没有被选为DR或BDR，但连接到广播网络或NBMA网络上的其他设备被选举为DR。它会与DR和BDR建立邻接关系。
- BDR：设备是相连的网络中的BDR，并将在当前的DR失效时成为DR。该设备与接入该网络的所有其他设备建立邻接关系。
- DR：设备是相连的网络中的DR。该设备与接入该网络的所有其他设备建立邻接关系。

#### OSPF邻居状态机

- Down：邻居会话的初始阶段。表明没有在邻居失效时间间隔内收到来自邻居设备的Hello报文。除了NBMA网络OSPF路由器会每隔PollInterval时间对外轮询发送Hello报文，包括向处于Down状态的邻居路由器（即失效的邻居路由器）发送之外，其他网络是不会向失效的邻居路由器发送Hello报文的。
- Init：本状态表示已经收到了邻居的Hello报文，但是对端并没有收到本端发送的Hello报文，收到的Hello报文的邻居列表并没有包含本端的Router ID，双向通信仍然没有建立。
- 2-Way：互为邻居。本状态表示双方互相收到了对端发送的Hello报文，报文中的邻居列表也包含本端的Router ID，邻居关系建立。如果不形成邻接关系则邻居状态机就停留在此状态，否则进入ExStart状态。DR和BDR只有在邻居状态处于2-Way及之后的状态才会被选举出来。
- ExStart：协商主从关系。建立主从关系主要是为了保证在后续的DD报文交换中能够有序的发送。邻居间从此时才开始正式建立邻接关系。
- Exchange：交换DD报文。本端设备将本地的LSDB用DD报文来描述，并发给邻居设备。
- Loading：正在同步LSDB。两端设备发送LSR报文向邻居请求对方的LSA，同步LSDB。
- Full：建立邻接。两端设备的LSDB已同步，本端设备和邻居设备建立了完全的邻接关系。

![](/images/Review-The-OSPF/ospf状态机.jpg)
### Ospf邻居建立过程

![](/images/Review-The-OSPF/Ospf邻居关系建立过程.png)

### Ospf运行机制

1. 通过交互Hello报文形成邻居关系

   路由器运行OSPF协议后，会从所有启动OSPF协议的接口上发送Hello报文。如果两台路由器共享一条公共数据链路，并且能够成功协商各自Hello报文中所指定的某些参数，就能形成邻居关系。

2. 通过泛洪LSA通告链路状态信息

   每台路由器根据自己周围的网络拓扑结构生成LSA，LSA描述了路由器所有的链路、接口、邻居及链路状态等信息，路由器通过交互这些链路信息来了解整个网络的拓扑信息。

3. 通过泛洪LSA通告链路状态信息

   通过LSA的泛洪，路由器会把收到的LSA汇总记录在LSDB中。

4. 通过SPF算法计算并形成路由

   当LSDB同步完成之后，每一台路由器都将以其自身为根，使用SPF算法来计算一个**无环路**的拓扑图来描述它所知道的到达每一个目的地的最短路径（最小的路径代价）。这个拓扑图就是最短路径树，有了这棵树，路由器就能知道到达自治系统中各个节点的最优路径。

5. 维护和更新路由表

   根据SPF算法得出最短路径树后，每台路由器将计算得出的最短路径加载到OSPF路由表形成指导数据转发的路由表项。

### Ospf组播地址

- 224.0.0.5 － 所有的OSPF路由器(OSPF 发送HELLO包)
- 224.0.0.6 － DR/BDR路由监听这个地址

### DR 与 BDR

- DR:路由器的核心者，称为指定路由器。邻居关系只会发给DR

- BDR：被指定路由器，如果DR失效，那么BDR就会顶上去工作

#### 为什么需要有DR？

在广播网络和NBMA网络中，任意两台路由器之间都要传递路由信息。若网络中有n台路由器，则需要建立n*(n-1)/2个邻接关系。这使得任何一台路由器的路由变化都会导致多次传递，浪费了带宽资源。

为解决这一问题，OSPF定义了DR。通过选举产生DR后，所有其他设备都只将信息发送给DR，由DR将网络链路状态LSA广播出去。

为了防止DR发生故障，重新选举DR时会造成业务中断，除了DR之外，还会选举一个备份指定路由器BDR。这样除DR和BDR之外的路由器（称为DR Other）之间将不再建立邻接关系，也不再交换任何路由信息，这样就减少了广播网和NBMA网络上各路由器之间邻接关系的数量。

#### DR和BDR的选举依据

比较优先级，优先级越大越优。默认优先级为1，如果优先级为0，则没有选举权。如果优先级一致，比较router-id，越大越优。

#### DR和BDR的选举原则

- 如果同一网段中的DR和BDR为空，首先选举BDR。将所有优先级大于0的设备放入到候选列表中（优先级为0的不参与选举），比较优先级，优先级越大越优。如果优先级一致，比较router-id，越大越优。BDR选举后，由BDR升级为DR并重新选举BDR。
- 如果同一网段中已有设备通告自己是BDR，但DR为空，则BDR升级为DR，重新选举BDR。
- 如果同一网段中已有设备通告自己是DR，但BDR为空，则选举BDR。
- 如果同一网段中只有唯一的一台设备通告自己为DR或BDR，则通告者为DR或BDR，DR或BDR角色不抢占。
- 如果在同一网段中存在多台设备同时通告自己为DR或BDR，则DR或者BDR需从新选举

##### 选举制

选举制是指DR和BDR不是人为指定的，而是由本网段中所有的路由器共同选举出来的。路由器接口的DR优先级决定了该接口在选举DR、BDR时所具有的资格，本网段内DR优先级大于0的路由器都可作为“候选人”。选举中使用的“选票”就是Hello报文，每台路由器将自己选出的DR写入Hello报文中，发给网段上的其他路由器。当处于同一网段的两台路由器同时宣布自己是DR时，DR优先级高者胜出。如果优先级相等，则Router ID大者胜出。如果一台路由器的优先级为0，则它不会被选举为DR或BDR

##### 终身制

终身制也叫非抢占制。每一台新加入的路由器并不急于参加选举，而是先考察一下本网段中是否已存在DR。如果目前网段中已经存在DR，即使本路由器的DR优先级比现有的DR还高，也不会再声称自己是DR，而是承认现有的DR。因为网段中的每台路由器都只和DR、BDR建立邻接关系，如果DR频繁更换，则会引起本网段内的所有路由器重新与新的DR、BDR建立邻接关系。这样会导致短时间内网段中有大量的OSPF协议报文在传输，降低网络的可用带宽。终身制有利于增加网络的稳定性、提高网络的可用带宽。实际上，在一个广播网络或NBMA网络上，最先启动的两台具有DR选举资格的路由器将成为DR和BDR

##### 继承制

继承制是指如果DR发生故障了，那么下一个当选为DR的一定是BDR，其他的路由器只能去竞选BDR的位置。这个原则可以保证DR的稳定，避免频繁地进行选举，并且DR是有备份的（BDR），一旦DR失效，可以立刻由BDR来承担DR的角色。由于DR和BDR的数据库是完全同步的，这样当DR故障后，BDR立即成为DR，履行DR的职责，而且邻接关系已经建立，所以从角色切换到承载业务的时间会很短。同时，在BDR成为新的DR之后，还会选举出一个新的BDR，虽然这个过程所需的时间比较长，但已经不会影响路由的计算了

### Ospf  LSA 类型

| LSA 类型                                                     | LSA 类型编号 | LAS用处                                                      |
| ------------------------------------------------------------ | ------------ | ------------------------------------------------------------ |
| Router LSA                                                   | 1            | 所有的OSPF speaker都会产生该类LSA，只在区域内传播，这种类型的LSA用于描述设备的链路状态和开销，在路由器所属的区域内传播 |
| Network LSA                                                  | 2            | 由ABR产生，描述区域内某个网段的路由，并通告给发布或接收此LSA的非Totally STUB或NSSA区域 |
| Network-summary-LSA                                          | 3            | 由ABR发布，用来描述区域间的路由信息。ABR将Network-summary-LSA发布到一个区域，通告该区域到其他区域的目的地址。实际上，ABR是将区域内部的Type1和Type2的信息收集起来并汇总之后扩散出去，这就是Summary的含义 |
| ASBR summary LSA                                             | 4            | 由ABR发布，描述到ASBR的路由信息，并通告给除ASBR所在区域的其他相关区域 |
| Autonomous system external LSA                               | 5            | 由ASBR产生，描述到AS外部的路由，通告到所有的区域（除了STUB区域和NSSA区域） |
| NSSA External LSA                                            | 7            | 由ASBR产生，描述到AS外部的路由，仅在NSSA区域内传播。NSSA区域的ABR收到NSSA LSA时，会有选择地将其转化为Type5 LSA，以便将外部路由信息通告到OSPF网络的其它区域。 |
| Opaque LSA（链路本地范围）、Opaque LSA（本地区域范围）、Opaque LSA（ AS 范围） | 9、10、11    | Opaque LSA提供用于OSPF的扩展的通用机制。其中：Type9 LSA仅在接口所在网段范围内传播。用于支持GR的Grace LSA就是Type9 LSA的一种。Type10 LSA在区域内传播。用于支持TE的LSA就是Type10 LSA的一种。Type11 LSA在自治域内传播，目前还没有实际应用的例子 |

### Ospf区域

#### 为什么需要划分区域

随着网络规模日益扩大，当一个大型网络中的路由器都运行OSPF路由协议时，会出现以下问题：

- 网络拓扑发生变化概率增大，LSA泛洪严重，降低网络带宽利用率。
- 路由器数量增多，LSDB庞大，占用大量存储空间，并使得运行SPF算法的复杂度增加。
- 每台路由器需要维护的路由表越来越大。

OSPF协议通过将自治系统划分成不同的区域，将LSA泛洪限制在一个区域内，提高网络的利用率和路由的收敛速率；每个区域内的路由器数量减少，维护的LSDB规模降低，SPF计算也仅限于区域内的LSA；每台路由器需要维护的路由表也越来越小。 此外，多区域提高了网络的扩展性，有利于组建大规模的网络。

区域是从逻辑上将路由器划分为不同的组，每个组用区域号（Area ID）来标识。区域的边界是路由器，而不是链路。一个网段（链路）只能属于一个区域，或者说每个运行OSPF的接口必须指明属于哪一个区域。

#### Ospf 路由器类型

- 区域内路由器（Internal Router）该类设备的所有接口都属于同一个OSPF区域。
- 区域边界路由器ABR（Area Border Router） 该类设备可以同时属于两个以上的区域，但其中一个必须是骨干区域。ABR用来连接骨干区域和非骨干区域，它与骨干区域之间既可以是物理连接，也可以是逻辑上的连接。
- 骨干路由器（Backbone Router）该类设备至少有一个接口属于骨干区域。所有的ABR和位于Area0的内部设备都是骨干路由器。
- 自治系统边界路由器ASBR（AS Boundary Router）与其他AS交换路由信息的设备称为ASBR。

ASBR并不一定位于AS的边界，它可能是区域内设备，也可能是ABR。只要一台OSPF设备引入了外部路由的信息，它就成为ASBR。

![](/images/Review-The-OSPF/Ospf 路由器类型.png)
#### 区域类型

- 普通区域，必须有且只能有一个。

  - 骨干区域：负责在非骨干区域之间发布由区域边界路由器汇总的路由信息（并非详细的链路状态信息），为避免区域间路由环路，非骨干区域之间不允许直接相互发布区域间路由。因此，所有区域边界路由器都至少有一个接口属于Area 0，即每个区域都必须连接到骨干区域。

  - 标准区域：最通用的区域，它传输区域内路由，区域间路由和外部路由。所有非骨干区域必须与骨干区域保持连通

- Stub 末梢区域：

  Stub区域的ABR不传播它们接收到的自治系统外部路由。

  一般情况下，Stub区域位于自治系统的边界，是只有一个ABR的非骨干区域，为保证到自治系统外的路由依旧可达，Stub区域的ABR将生成一条缺省路由，并发布给Stub区域中的其他非ABR路由器

  - 骨干区域不能配置成Stub区域。
  - 如果要将一个区域配置成Stub区域，则该区域中的所有路由器都要配置Stub区域属性。
  - Stub区域内不能存在ASBR，因此自治系统外部的路由不能在本区域内传播。
  - 虚连接不能穿过Stub区域。

- Total Stub 绝对末梢区域：允许ABR发布Type3缺省路由，不允许发布自治系统外部路由和区域间的路由，只允许发布区域内路由。

- NSSA 次末梢区域：

  不允许存在Type5 LSA。

  NSSA区域允许引入自治系统外部路由，携带这些外部路由信息的Type7 LSA由NSSA的ASBR产生，仅在本NSSA内传播。

  当Type7 LSA到达NSSA的ABR时，由ABR将Type7 LSA转换成Type5 LSA，泛洪到整个OSPF域中。

  - 骨干区域不能配置成NSSA区域。
  - 如果要将一个区域配置成NSSA区域，则该区域中的所有路由器都要配置NSSA区域属性。
  - NSSA区域的ABR会发布Type7 LSA缺省路由传播到本区域内。
  - 所有域间路由都必须通过ABR才能发布。
  - 虚连接不能穿过NSSA区域。

- Total NSSA 绝对次末梢区域：不允许发布自治系统外部路由和区域间的路由，只允许发布区域内路由。

### Ospf区域间环路及防环方法

Ospf在区域内部运行的是SPF算法，这个算法能够保证区域内部的路由不会成环。然而划分区域后，区域之间的路由传递实际上是一种类似距离矢量算法的方式，这种方式容易产生环路。

为了避免区域间的环路，Ospf规定直接在两个非骨干区域之间发布路由信息是不允许的，只允许在一个区域内部或者在骨干区域和非骨干区域之间发布路由信息。因此，每个ABR都必须连接到骨干区域。

假设Ospf允许非骨干区域之间直接传递路由，则可能会导致区域间环路。如下图所示，骨干区连接到其他网络的路由信息会传递至Area 1。假设非骨干区之间允许直接传递路由信息，那么这条路由信息最终又被传递回去，形成区域间的路由环路。为了防止这种区域间环路，OSPF禁止Area 1和Area 3，以及Area 2和Area 3之间直接进行路由交互，而必须通过骨干区域进行路由交互。这样就能防止区域间环路的产生。

![](/images/Review-The-OSPF/Ospf区域间环路.png)
### Ospf 虚连接

虚连接（Virtual link）是指在两台ABR之间通过一个非骨干区域建立的一条逻辑上的连接通道。

根据RFC 2328，在部署OSPF时，要求所有的非骨干区域与骨干区域相连，否则会出现有的区域不可达的问题。但是在实际应用中，可能会因为各方面条件的限制，无法满足所有非骨干区域与骨干区域保持连通的要求，此时可以通过配置OSPF虚连接来解决这个问题。

通过虚连接，两台ABR之间直接传递OSPF报文信息，两者之间的OSPF设备只是起到一个转发报文的作用。由于OSPF协议报文的目的地址不是这些设备，所以这些报文对于两者而言是透明的，只是当作普通的IP报文来转发。

![](/images/Review-The-OSPF/Ospf非骨干区没有连接骨干区.png)

![](/images/Review-The-OSPF/Ospf非骨干区没有连接骨干区.png)

### Ospf路由聚合和路由过滤

#### Ospf路由聚合

路由聚合是指ABR可以将具有相同前缀的路由信息聚合到一起，只发布一条路由到其它区域。

通过路由聚合，可以减少路由信息，从而减小路由表的规模，提高设备的性能。

OSPF有两种路由聚合方式：

- 区域间路由聚合

  区域间路由聚合在ABR上完成，主要用于聚合AS内区域之间的路由。ABR向其它区域发送路由信息时，以网段为单位生成Type3 LSA。如果该区域中存在一些连续的网段，ABR可以将这些连续的网段聚合成一个网段。这样ABR只发送一条聚合后的LSA，所有属于命令指定的聚合网段范围的LSA将不会再被单独发送出去。

- 外部路由聚合

  外部路由聚合在ASBR上完成，主要用于聚合OSPF引入的外部路由。ASBR将对引入的聚合地址范围内的Type5 LSA进行聚合。当配置了NSSA区域时，还要对引入的聚合地址范围内的Type7 LSA进行聚合。

  如果本地设备既是ASBR又是ABR，则对由Type7 LSA转化成的Type5 LSA进行聚合处理。

#### Ospf路由过滤

OSPF支持使用路由策略对路由信息进行过滤。缺省情况下，OSPF不进行路由过滤。

OSPF可以使用的路由策略包括route-policy、访问控制列表（access-list）和地址前缀列表（prefix-list）。

### Ospf多进程

OSPF支持多进程，在同一台路由器上可以运行多个不同的OSPF进程，它们之间互不影响，彼此独立。不同OSPF进程之间的路由交互相当于不同路由协议之间的路由交互。

路由器的一个接口只能属于某一个OSPF进程。

OSPF多进程的一个典型应用就是在VPN场景中PE和CE之间运行OSPF协议，同时VPN骨干网上的IGP也采用OSPF。在PE上，这两个OSPF进程互不影响。

### Ospf GTSM

### Ospf GR

### Ospf 邻居震荡抑制

### Ospf TE