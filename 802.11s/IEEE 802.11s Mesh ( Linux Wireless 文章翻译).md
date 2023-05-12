# IEEE 802.11s Mesh ( Linux Wireless 文章翻译 )

> 原文链接：[en:developers:documentation:ieee80211:802.11s [Linux Wireless\] (kernel.org)](https://wireless.wiki.kernel.org/en/developers/documentation/ieee80211/802.11s)

## 简介

IEEE 802.11s 标准是 IEEE 802.11 标准的一个扩展，802.11s 允许多个无线节点之间直接互相连接，而无需再在中间添加新的 AP 。这里举一个简单的例子：比如你家里有一个 AP ，有两个笔记本 A 和 B 都连接到了这个 AP 上，那么笔记本 A 拷贝一个文件到 B 的过程就是这样的：A 将数据传输到 AP ，然后 AP 再将这些数据传输给 B 。在该场景中，A 和 B 之间所有的通信都是经过 AP 进行的。

IEEE 802.11s 则允许 A 和 B 在没有 AP 的情况下直接进行通信，而这并非 IEEE 802.11s 的最大用处 —— 无线多节点场景才是 IEEE 802.11s 真正大显身手的地方。通过 802.11s mesh 标准，网络节点能够在全无线的情况下组成多跳网络（ multi-hop network ），这就意味着建立网络不再需要有线的基础设施了，这在很多场景下都能带来更便捷的体验——例如，一个救灾队伍中的多个支持 IEEE 802.11s 的笔记本电脑之间，即使无法直接相连，也能通过中间的其他节点实现通信。

IEEE 802.11s 标准定义了一个**无线分布式系统（Wireless Distribution System, WDS）** 。在大学中就有 WDS 具体实现的例子：校园里中一般都会装很多的 AP ，这些 AP 都有着相同的 SSID 。AP 通过 Ethernet 这类有线网络实现互联，用户能够无线连接到 AP 。802.11s 就相当于是把上面这个场景中连接各个 AP 的物理电缆换成了无线连接。这样就能够省下布线及维护等基础设施的开销，也提高了移动 AP 或添加新 AP 的灵活性。

## 术语

IEEE 802.11s 定义了下面三类节点：

- Mesh Point ( MP )
- Mesh Portal ( MPP )
- Mesh Access Point ( MAP )

所有的节点都能够进行帧转发，进而在网络中的任意两个点之间传递帧

### Mesh Point

Mesh Point ( MP ) 支持 Peer Link Management 协议，该协议用于发现附近的节点并对它们进行追踪。需要注意的是，附近节点发现的范围限制在 MP 的范围之内。

对于距离超过一跳的节点间通信，MP 提供了混合无线 Mesh 协议（ Hybrid Wireless Mesh Protocol, HWMP ）。该协议被称为 "混合" 是因为其支持两种路径选择协议（ path selection protocol ）。尽管这些协议看起来和路由协议（ routing protocols ）非常像，但请记住 IEEE 802.11s 是使用 MAC 地址，而非 IP 地址进行 "路由" 的，因此，这里使用一个专门的术语 "路径(path)" 来替代 "路由(route)"

### Mesh Portal

一个 IEEE 802.11s mesh 网络可以应用在很多方面，其中一个就是提供廉价的互联网接入服务。在该场景下，可以有一到多个 mesh 节点连入互联网。连接到 Mesh 网络的用户能够通过被称为 "Mesh Portals" ( MPP ) 的网关节点访问互联网。MPP 节点同时连接到 mesh 网络和互联网，它是 mesh 网络到互联网的网关。

需要注意的是，一个 MPP 必须桥接至少两个 interface ，才能提供网关的功能

### Mesh Access Point

MAP 是增加了 mesh 功能的传统 AP 。其能够作为 AP 提供服务，同时也是 mesh 网络的一部分。

### Peer Link Management Protocol, PLMP

Peer Link Management Protocol 是用于对邻居节点进行发现和维护的协议。当一个无线节点被配置为 MP 时，它就会开始传输信标帧（ beacons ）。其他在该节点范围内的节点在接收到新节点的信标帧时，就会开始初始化 Peer Link Management Protocol FSM

> 译注：FSM 是 Finite State Machine， 有限状态机

### 混合无线 mesh 协议（ Hybrid Wireless Mesh Protocol, HWMP ）

HWMP 用于进行路径（ path ）选择，用以寻找到达非直接邻居节点的"路由"（ route ）。HWMP 主要包含两部分，其一是先验式路由模式（ Proactive portion ），其二是按需路由模式（ on-demand portion ）。前者基本上是一个基于树的层级路由协议（ tree-based hierarchical routing protocol ），后者是 Ad-hoc 按需向量的修改版本（ Ad-hoc On Demand Vector, AODV ）

> 译注：HWMP 的先验式路由模式通过定期发送控制帧维护一个树状拓扑。按需路由模式在需要时发送请求帧和回复帧来发现最佳的路径

### Information Elements

IEEE 802.11 支持信标帧（ beacon ）或探针响应帧（ probe response frame ）中包含的信息元素（ Information Element, IE ）。IE 有多个种类，IEEE 802.11s 使用了下面这些 IE ：

- Mesh 配置
- Peer Link Information Element
- 路径请求（Path Request, PREQ），路径重播（Path Reply, PREP），路径错误（Path Error, PERR），根节点通告（Root Annoucement, RANN）（ HWMP 特有）。上面的列表可能并不完整

> 译注：
>
> IE 由一个元素 ID、一个长度和一个信息字段组成，不同的元素 ID 对应不同的信息类型
>
> 探针响应帧是一种管理帧，用于响应探针请求帧（Probe request frame）的请求。其包含的信息与信标帧基本一致，如 SSID、支持的数据速率、加密类型等，用于同步和配置客户端、接入点之间的连接
>
> root announcement 是一种在 mesh 网络中选举和通告 root 节点的机制。root 节点是一个特殊的 mesh 节点，其可以与外部网络通信，并为其他 mesh 节点提供网关服务

### 网络建立（Network Formation）

*iw* 工具可以用于将节点配置为 MP 。配置至少要包含设置 mesh ID （类似基于基础设施（AP）的网络中的 SSID ）和信道（channel）。配置中的其他内容由 Mesh 配置信息元素（Mesh Configuration Information Element）描述。一旦节点被配置为 MP ，其就会开始以一定的时间间隔（默认是每秒）发出信标帧。这些信标帧会包含上面提到的 Mesh 配置信息元素 。该信息元素包含下列字段：

- 路径选择协议 ID （Path Selection Protocol ID）

- 路径选择 metric （Path Selection metric）

  > 译注：path selection metric 用于选择最佳路径。metric 是一种度量标准，用于比较不同路径的优劣。不同的协议有不同的 metric，例如 RIP 使用跳数，OSPF 使用代价，EIGRP 使用带宽和延迟等。

- 拥塞控制模式

- 同步协议 ID

- 认证协议 ID

- Mesh 建立信息（Mesh Formation Info, 通常是邻居 / peer 的数量）

- Mesh 能力（Mesh Capability, 包含该节点是否允许建立 peer 链接的信息）对于 mac80211，mesh 接口表示为如下所示的 ieee80211_if_mesh 结构体：

```c
struct ieee80211_if_mesh {
        struct work_struct work;
        struct timer_list housekeeping_timer;
        struct timer_list mesh_path_timer;
        struct timer_list mesh_path_root_timer;
        struct sk_buff_head skb_queue;
     
        unsigned long timers_running;
     
        unsigned long wrkq_flags;
     
        u8 mesh_id[IEEE11_MAX_MESH_ID_LEN];
        size_t mesh_id_len;
        /* Active Path Selection Protocol Identifier */
        u8 mesh_pp_id;
        /* Active Path Selection Metric Identifier */
        u8 mesh_pm_id;
        /* Congestion Control Mode Identifier */
        u8 mesh_cc_id;
        /* Synchronization Protocol Identifier */
        u8 mesh_sp_id;
        /* Authentication Protocol Identifier */
        u8 mesh_auth_id;
        /* Local mesh Sequence Number */
        u32 sn; 
        /* Last used PREQ ID */
        u32 preq_id;
        atomic_t mpaths; 
        /* Timestamp of last SN update */
        unsigned long last_sn_update;
        /* Timestamp of last SN sent */ 
        unsigned long last_preq;
        struct mesh_rmc *rmc;
        spinlock_t mesh_preq_queue_lock;
        struct mesh_preq_queue preq_queue;
        int preq_queue_len;
        struct mesh_stats mshstats;
        struct mesh_config mshcfg; 
        u32 mesh_seqnum;
        bool accepting_plinks;
};
```

上面的结构体维护一个 mesh 接口的状态信息。mesh 接口的当前配置信息就保存在该接口中。

Mesh 配置信息元素非常重要。若一个节点被配置为连接到与某个节点相同的网络，或其已经是【已有的、用相同 Mesh 配置信息元素配置的网络】的一部分，则该节点应该在接收到已有节点的信标帧时立即处理该帧。检查过程由 `mesh_matches_local` 函数执行。

希望加入已有 mesh 网络的新节点必须使用正确的 mesh ID 进行配置，且必须设置正确的信道。一旦完成，新节点就会开始发送信标帧。此时会发生两件事：启动网络的节点会从新节点接收信标帧，新节点也会开始接收来自已有节点的信标帧。

当接收到节点时，两个节点就会启动 Peer 链接管理协议，主要会进行一个四步握手。握手完成后，每个节点就会将对方加入到自己的 station hash 中（即"邻居表", neighbor table ）。下面的字符图描述了理想状况下（即没有损失）的四步握手。图中的 P2 是 mesh 网络中的已有节点，P1 是接收到来自 P2 的信标帧或探针响应帧的节点。P1 可以是要加入网络的新节点，也可以是失去连接的已有节点（由于位置移动、崩溃等原因失去连接），总之现在可以再次收到 P2 的信标帧了。

```
                 Beacon/Probe resp
P1's state       <--------------       P2's state
PLINK_LISTEN                         PLINK_LISTEN
                 PLINK_OPEN
PLINK_OPN_SNT    -------------->     PLINK_OPN_RCVD
                                                   
                 PLINK_OPEN
PLINK_OPN_RCVD   <--------------     PLINK_OPN_RCVD

                 PLINK_CONFIRM
PLINK_ESTAB      <--------------     PLINK_OPN_RCVD

                 PLINK_CONFIRM
PLINK_ESTAB      --------------->    PLINK_ESTAB
```

所有节点连接管理帧（ peer link management frames ）都是 IEEE 802.11 管理类型帧，子类型是 action 。用于处理 action 类型的 802.11s ( mesh ) 管理帧的函数是 `ieee80211_mesh_rx_queued_mgmt` （由 `ieee80211_iface_work` 调用，该函数是接口工作队列函数）。下面是该函数的一部分代码：

```C
switch (stype) {
    case IEEE80211_STYPE_PROBE_RESP:
    case IEEE80211_STYPE_BEACON:
        ieee80211_mesh_rx_bcn_presp(sdata, stype, mgmt, skb->len,
                                    rx_status);
        break;
    case IEEE80211_STYPE_ACTION:
        ieee80211_mesh_rx_mgmt_action(sdata, mgmt, skb->len, rx_status);
        break;
}
```

上面的 `stype` 是收到的 IEEE 802.11 帧的子类型。你可以分一步分析代码来找到更多关于节点连接管理状态机（ peer link management state machine ）的信息。

注意，HWMP 帧也是 *action* 子类型的管理帧。因此你在这里也能找到 ieee80211_mesh_rx_mgmt 相关的信息。

HWMP 的按需模式与 AODV 非常像，所以建议读者阅读更多关于 AODV 的信息来熟悉代码。

