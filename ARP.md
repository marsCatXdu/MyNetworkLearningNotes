# ARP

ARP - Address Resolution Protocol. 其功能是从 IP 地址获取物理地址（MAC）。一共有三个与 ARP 相关的术语——Reverse ARP, Proxy ARP, Inverse ARP

- Reverse ARP: LAN 中的 client 向路由器请求自己的 IP 地址。目前该协议已不再被广泛使用—— DHCP 接替了它分配 IP 地址的工作

- Proxy ARP: 它能让连接到同一路由器上的、不同网段的设备将 IP 地址解析为 MAC 地址。

  > 例如 A 想要和另一网络中的 B 通信，但 A 并没有配置默认的网关，那么 A 就会向 B 的 IP 地址发送 ARP 请求。连接 A、B 各自所在网络的路由器会接收到这个 ARP 请求，并用自己的 MAC 地址回复，假扮自己就是 B。这样一来，A 就可以发送发往路由器 MAC 地址的数据包了，然后路由器就可以将这样的数据包转发给 B 了。这样的好处之一是 A 甚至不需要知道 B 在一个路由器后面

- Inverse ARP: 使用 MAC 查找 IP 地址。在 ATM （Asynchronous Transfer Mode）网络中默认使用 Inverse ARP

