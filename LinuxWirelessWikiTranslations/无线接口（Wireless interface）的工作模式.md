无线接口有下面几种工作模式，工作模式决定无线链路的主要功能。

## 接入点基础设施模式 - AccessPoint (AP) Infrastructure mode

AP 在被管理无线网络（managed wireless network）中扮演主（Master）设备的角色。AP 通过管理和维护关联站点（Stations）列表的方式来维持网络，同时也管理安全策略。该网络使用 AP 的 MAC 地址（BSSID）命名。人类可读的网络名称（SSID）也由 AP 设置

## 站点基础设施模式 - Station infrastructure mode

站点设备通过发送特定的管理数据包给 AP 来与 AP 建立连接。这个建立连接的过程被称为 认证（authentication） 和 关联（association）。当 AP 发出关联成功（successful）回复后，站点就成为网络中的一员了。

该模式在无线扩展工具（例如 `iwconfig` ）中也被称为"被管理(managed)"。

## 监控模式 - Monitor mode

监控模式是一个仅被动模式（passive-only mode），不会传输数据包。所有发来的数据包都会被不经过滤地直接转发给所在的主机。该模式可用于观察网络的状况。

通过 mac80211，可以实现让一个网络设备在以监控模式运行的同时，还能作为普通设备使用。然而并不是所有的硬件都完全支持该模式，监控模式接口总是提供 "best effort" 的服务。

mac80211 还可以实现在监控模式下发出数据包，这被称为"包注入（packet injection）"。这对于想实现在用户空间（userspace）工作的 MLME 的应用来说很有用，比如支持 IEEE 802.11 的非标准 MAC 扩展。

参阅：[radiotap](https://wireless.wiki.kernel.org/en/developers/documentation/radiotap)

## Ad-Hoc (IBSS) 模式

Ad-Hoc 模式被用于在不需要 Master AP 的情况下创建无线网络。在 IBSS 中的每个站点都自行维护网络。Ad-Hoc 模式可以在没有（可用）AP 的情况下连接两个或多个计算机。

## 无线分发系统 - Wireless Distribution System (WDS)

Distribution System 是 AP 的有线上行连接，无线分发系统则是它的无线版。WDS 作为无线通信路径，为与之连接的 AP 协作（通常在一个单独的 ESS 中）（ESS = Extended Service Set），其可以用来替代有线连接。关于如何启用该模式，请参阅 [iw WDS](https://wireless.wiki.kernel.org/en/users/documentation/iw) 文档，但也可以考虑使用 [4地址模式](https://wireless.wiki.kernel.org/en/users/documentation/iw#using_4-address_for_ap_and_client_mode)

> 服务集：[Service set (802.11 network) - Wikipedia](https://en.wikipedia.org/wiki/Service_set_(802.11_network))

## Mesh

Mesh 接口用于让多个设备通过建立动态的智能路由进行通信。参阅：[IEEE 802.11s - Wikipedia](https://en.wikipedia.org/wiki/IEEE_802.11s)， [Wireless mesh network - Wikipedia](https://en.wikipedia.org/wiki/Wireless_mesh_network)

可以通过桥接 mesh 接口和普通以太网接口的方式，实现 mesh portal 功能