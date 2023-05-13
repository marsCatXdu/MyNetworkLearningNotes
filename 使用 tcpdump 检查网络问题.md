# 使用 tcpdump 检查网络问题

> 基础翻译、删改自 [Troubleshoot your network with tcpdump | Enable Sysadmin (redhat.com)](https://www.redhat.com/sysadmin/troubleshoot-tcpdump)，原作者 Tyler Carrigan
>
> 补充内容：[An introduction to using tcpdump at the Linux command line | Opensource.com](https://opensource.com/article/18/10/introduction-tcpdump)
>
> [Packet sniffer basics for network troubleshooting | Enable Sysadmin (redhat.com)](https://www.redhat.com/sysadmin/packet-sniffer-basics)

## 什么是 tcpdump ？

`tcpdump` 是一个诞生自 1980 年代的网络检查工具。从最基本的层面来说，`tcpdump` 是一个用于检查网络连接问题的数据包捕捉工具。其可能是最接近于 wireshark 的工具，但要轻量很多且只能在命令行使用

## 基本用法

先来看看最基本的功能。如果想要开始捕获一个接口（interface）的软件包，需要先看看可以捕获数据的接口：

```
$ sudo tcpdump -D
```

输出示例如下

```
[tcarrigan@server ~]$ sudo tcpdump -D
[sudo] password for tcarrigan: 
1.enp0s3 [Up, Running]
2.enp0s8 [Up, Running]
3.lo [Up, Running, Loopback]
4.any (Pseudo-device that captures on all interfaces) [Up, Running]
5.virbr0 [Up]
6.bluetooth-monitor (Bluetooth Linux Monitor) [none]
7.nflog (Linux netfilter log (NFLOG) interface) [none]
8.nfqueue (Linux netfilter queue (NFQUEUE) interface) [none]
9.usbmon0 (All USB buses) [none]
10.usbmon1 (USB bus number 1)
11.virbr0-nic [none]
```

在命令在企业场景下极为有用，因为经常会用特定的接口来移动特定类型的数据。下面捕捉一些数据包，然后看看能在输出中找到什么信息。

基本的捕获命令如下：

```
[root@server ~]# tcpdump -i any
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
18:42:10.914742 IP server.example.com.55018 > 216.126.233.109.ntp: NTPv4, Client, length 48
18:42:10.915759 IP server.example.com.59656 > router.charter.net.domain: 1974+ PTR? 109.233.126.216.in-addr.arpa. (46)
18:42:10.959920 IP router.charter.net.domain > server.example.com.59656: 1974 ServFail 0/0/0 (46)
18:42:10.960089 IP server.example.com.42825 > router.charter.net.domain: 1974+ PTR? 109.233.126.216.in-addr.arpa. (46)
*** Shortened output ***
^C
17 packets captured
18 packets received by filter
1 packet dropped by kernel
```

这里使用了 `-i` 来指定要监听的接口 `any` 。tcpdump 会一直捕捉数据包，直到收到 Ctrl+C 发来的中断信号。另外还可以使用 `-c` 来指定要捕获的数据包数量

```
[root@server ~]# tcpdump -i any -c 3
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
18:51:54.509439 IP server.example.com.58249 > 216.126.233.109.ntp: NTPv4, Client, length 48
18:51:54.510413 IP server.example.com.46277 > router.charter.net.domain: 9710+ PTR? 109.233.126.216.in-addr.arpa. (46)
18:51:54.570112 IP 216.126.233.109.ntp > server.example.com.58249: NTPv4, Server, length 48
3 packets captured
10 packets received by filter
1 packet dropped by kernel
```

默认情况下，tcpdump 会将 IP 地址和端口解析为名称。我们可以禁用掉解析来让 tcpdump 直接输出 IP 地址和端口号。使用 `-nn` 来关闭名称与端口解析：

```
[root@server ~]# tcpdump -i any -c3 -nn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
19:56:12.804327 IP 10.0.3.15.41153 > 64.79.100.196.123: NTPv4, Client, length 48
19:56:12.867789 IP 64.79.100.196.123 > 10.0.3.15.41153: NTPv4, Server, length 48
19:56:13.739885 IP 10.0.3.15.50968 > 216.126.233.109.123: NTPv4, Client, length 48
3 packets captured
3 packets received by filter
0 packets dropped by kernel
```

## 其他有用的过滤器

按 IP 地址过滤：

```
$ sudo tcpdump host x.x.x.x
```

按接口过滤：

```
$ sudo tcpdump eth0
```

按源过滤：

```
$ sudo tcpdump src x.x.x.x
```

按目的地过滤：

```
$ sudo tcpdump dst x.x.x.x
```

按协议过滤：

```
$ sudo tcpdump icmp
```

还有很多其他的选项和过滤器，能够帮助你更好地过滤内容。若有需要请参考更多资料

## 实际应用

当我还是一个支持工程师的时候，我花费了很多的时间来解决将数据从生产服务器复制到灾备环境中时的问题。客户经常会设置特定的复制接口（interface）来从生产服务器发送流量到备份目标服务器。我们先了解一下这个问题，然后使用 tcpdump 来验证从源接口到目的地的流量。

### 已有条件

- 源服务器 - 172.25.1.5
- 目的地服务器 - 172.25.1.4
- 复制接口 - enp0s8

理论上，当启动数据复制任务时，应该能观察到从 172.25.1.5 到 172.25.1.4 的流量

我在源服务器上启动了一个简单的 "复制"（`ping`）任务。然后在源服务器和目的地服务器上使用 `tcpdump` 来检查是否能收到流量。

源：

```
[root@server ~]# tcpdump -i enp0s8 -nn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
23:17:55.347648 ARP, Request who-has 172.25.1.4 tell 172.25.1.5, length 28
23:17:56.378194 ARP, Request who-has 172.25.1.4 tell 172.25.1.5, length 28
23:17:57.398294 ARP, Request who-has 172.25.1.4 tell 172.25.1.5, length 28
23:17:58.422946 ARP, Request who-has 172.25.1.4 tell 172.25.1.5, length 28
23:17:59.448412 ARP, Request who-has 172.25.1.4 tell 172.25.1.5, length 28
^C
5 packets captured
5 packets received by filter
0 packets dropped by kernel
```

从上面的输出可以看出，流量只有请求，而没有来自目标的响应。在真实场景下，这表示目的地可能存在问题，因为我们可以看到流量已经从源接口发出去了。

然后我打开目的地的接口，表示已经解决问题

下面是在问题解决后，在源端捕捉到的流量

源：

```
[root@server ~]# tcpdump -i enp0s8 -c3 -nn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
23:22:04.694919 IP 172.25.1.5 > 172.25.1.4: ICMP echo request, id 7168, seq 911, length 64
23:22:04.695346 IP 172.25.1.4 > 172.25.1.5: ICMP echo reply, id 7168, seq 911, length 64
23:22:05.724968 IP 172.25.1.5 > 172.25.1.4: ICMP echo request, id 7168, seq 912, length 64
3 packets captured
3 packets received by filter
0 packets dropped by kernel
```

目的地：

```
[root@client ~]# tcpdump -i enp0s8 -c3 -nn
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
23:22:13.916519 IP 172.25.1.5 > 172.25.1.4: ICMP echo request, id 7168, seq 920, length 64
23:22:13.916569 IP 172.25.1.4 > 172.25.1.5: ICMP echo reply, id 7168, seq 920, length 64
23:22:14.935720 IP 172.25.1.5 > 172.25.1.4: ICMP echo request, id 7168, seq 921, length 64
3 packets captured
4 packets received by filter
0 packets dropped by kernel
```



## 输出格式

tcpdump 捕捉到的典型 TCP 包差不多是这样的：

```
08:41:13.729687 IP 192.168.64.28.22 > 192.168.64.1.41916: Flags [P.], seq 196:568, ack 1, win 309, options [nop,nop,TS val 117964079 ecr 816509256], length 372
```

第一个字段是根据本地时钟确定的收到包的时间戳

第二个字段 IP 表示这个包属于网络层协议

下一个字段 `192.168.64.28.22` 表示源的地址和端口，然后是目的地的地址和端口

在源和目的地后，是 TCP Flags `Flags [P.]`. 该字段的典型值包括：

| 值   | Flag 类型 | 描述                   |
| ---- | --------- | ---------------------- |
| S    | SYN       | 连接开始               |
| F    | FIN       | 连接结束               |
| P    | PUSH      | 数据推送               |
| R    | RST       | 连接重置（reset）      |
| .    | ACK       | 确认（Acknowledgment） |

这个字段可以包含多个值，比如 `[S.]` 代表 `SYN-ACK` 数据包

接下来是数据包所包含的数据的序列数（sequence number）。第一个捕捉到的包的这个数字是一个绝对数字，后面的为了方便跟踪则都是相对数字。在这个例子中，序列为 `seq 196:568` 代表这个数据包包含着本数据流的第 196 到 568 字节。

后面是一个确认数：`ack 1`。在本例中，因为这是发送方的数据所以该值为 1 。在数据的接收方，该值代表的是在这个流上期望的下一字节的数据。比如该流中下一数据包的 Ack number 应该是 568。

下一个是窗口大小 `win 309` ，表示接收缓冲区可用的字节数，然后是 TCP 选项例如 MSS （Maximum Segment Size ，最大端尺寸）或 Window Scale。关于 TCP 协议选项的细节，参阅 [Transmission Control Protocol (TCP) Parameters (iana.org)](https://www.iana.org/assignments/tcp-parameters/tcp-parameters.xhtml)

最后是数据包长度 `length 372` ，代表 payload 数据的字节长度。该长度是序列数中两个数字的差值



## 检查数据包内容

上面都只检查了数据包的头。但有时还需要检查数据包的内容。可以使用 `-X` 来以十六进制格式打印内容，使用 `-A` 来以 ASCII 格式打印内容

例：

```
# tcpdump -i ens9 -c 1 -X port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens9, link-type EN10MB (Ethernet), capture size 262144 bytes
21:27:47.269762 IP localhost.localdomain.38787 > dns.google.domain: 48705+ [1au] A? www.google.com. (43)
      0x0000: 4500 0047 0a9c 0000 4011 24b5 c0a8 7a9d E..G....@.$...z.
      0x0010: 0808 0808 9783 0035 0033 a65f be41 0120 .......5.3._.A..
      0x0020: 0001 0000 0000 0001 0377 7777 0667 6f6f .........www.goo
      0x0030: 676c 6503 636f 6d00 0001 0001 0000 2910 gle.com.......).
      0x0040: 0000 0000 0000 00 .......
1 packet captured
5 packets received by filter
0 packets dropped by kernel
```

