# iw

https://wireless.wiki.kernel.org/en/users/documentation/iw

iw 是一个基于 [nl80211](https://wireless.wiki.kernel.org/en/developers/documentation/nl80211) 的无线设备 CLI 配置工具。其支持所有最近添加进内核的新驱动。之前使用 Wireless Extensions 接口的 iwconfig 已经过时，强烈推荐换为使用 iw 。

和 Linux 内核的其他部分一样，iw 仍处于开发阶段，不断添加新的特性。

关于如何使用 iw 替换 iwconfig，请参阅：https://wireless.wiki.kernel.org/en/users/documentation/iw/replace-iwconfig



## 获取 iw

可以从这里获取发布的 tar 包：http://kernel.org/pub/software/network/iw/

也可以从 git 下载：http://git.kernel.org/?p=linux/kernel/git/jberg/iw.git



## 构建需求

- libnl >= libnl1
- libnl-dev >= libnl-dev-1
- pkg-config

需要有 libnl 才能使用 iw。第一个能用的版本是 1.0 pre8 ，该版本引入了 genl - Generic Netlink，其是 nl80211 的依赖。若你正在使用的发行版没有这个版本，则需要自行编译 libnl



## 帮助

```
iw help
```



## 获取设备功能（capabilities）

用下面的命令能够获取所有设备的设备功能信息

```
iw list
```



## 扫描

```
iw dev wlan0 scan
```



## 监听事件

```
iw event
```

在调试时查看 auth/assoc/deauth/disassoc 帧：

```
iw event -f
```

获取计时信息：

```
iw event -t
```



## 获取链路状态

要确定是否连接到了某个 AP ，或者要获取最近的 TX rate，可以用下面的命令。

当关联到经典 AP（非 802.11n ）时的输出样例：

```
iw dev wlan0 link
Connected to 04:21:b0:e8:c8:8b (on wlan0)
        SSID: attwifi
        freq: 2437
        RX: 2272 bytes (18 packets)
        TX: 232 bytes (3 packets)
        signal: -57 dBm
        tx bitrate: 36.0 MBit/s
```

当关联到 802.11n AP 时的输出样例：

```
iw dev wlan0 link
Connected to 68:7f:74:3b:b0:01 (on wlan0)
        SSID: tesla-5g-bcm
        freq: 5745
        RX: 30206 bytes (201 packets)
        TX: 4084 bytes (23 packets)
        signal: -31 dBm
        tx bitrate: 300.0 MBit/s MCS 15 40Mhz short GI
```

当没有连接到 AP 时的输出样例：

```
iw dev wlan0 link
Not connected.
```

如果要连接的 AP 没有加密，或使用 WEP 加密，则可以用 `iw connect` 连接 AP。

如果 AP 使用 WPA 或 WPA2 加密，则必须使用 [wpa_supplicant](https://wireless.wiki.kernel.org/en/users/documentation/wpa_supplicant) 进行连接。



## 建立基本连接

当且仅当 AP 满足下列条件时，才能直接使用 `iw` 进行连接

- 无加密
- 使用 WEP 加密

注意：如果与 AP 断开连接了（这在无线电环境繁忙的环境中还挺常见），你需要重新执行一遍这个命令才能重新连接。如果不想这么麻烦，那么就用 [wpa_supplicant](https://wireless.wiki.kernel.org/en/users/documentation/wpa_supplicant) 进行连接吧，该工具能够在连接掉线时尝试自动重连

如果要连接一个 SSID 是 `foo` 的无加密 AP，使用下面的命令：

```
iw wlan0 connect foo
```

如果有两个 AP 的 SSID 都是 `foo` ，而且你知道你想连接的那个工作频率为 2432 ，那就可以用下面这条命令指定要连接的 AP 的频率：

```
iw wlan0 connect foo 2432
```

要连接使用 WEP 的 AP，使用如下命令：

```
iw wlan0 connect foo keys 0:abcde d:1:0011223344
```



## 获取站点统计信息

要获取诸如 tx/rx 字节数量、上一个 TX 比特率（包含 MCS 率）等站点统计信息，可以用如下命令：

```
$ iw dev wlan1 station dump
Station 12:34:56:78:9a:bc (on wlan0)
        inactive time:  304 ms
        rx bytes:       18816
        rx packets:     75
        tx bytes:       5386
        tx packets:     21
        signal:         -29 dBm
        tx bitrate:     54.0 MBit/s
```



## 获取与特定 peer 的站点统计信息

如果想要获取站点正在通信的某个 peer 的统计信息，可以使用如下命令：

```
sudo iw dev wlan1 station get <peer-MAC-address>
```



## 修改传输比特率

iw 支持修改 TX 比特率，legacy 和 HT MCS rates。iw 是通过屏蔽允许的比特率（masking in the allowed bitrates）实现的这一点，iw 也允许你清除掩码。

### 修改 tx legacy bitrates

可以设置传输偏好，要求只使用几个 legacy bitrates。比如：

```
iw wlan0 set bitrates legacy=2.4 12 18 24
```

下面是启用某些人称之为 "Purge G" 的东西的方法。"Purge G" 关闭了 802.11b 关联

```
iw wlan0 set bitrates legacy-2.4 6 12 24
```

### 修改 tx HT MCS bitrates

iw 通过允许用户指定 band 和 MCS 率，实现了对使用 MCS 率的传输设置。需要注意的是，设备是否真正按照你的要求执行，取决于设备驱动和固件。用法如下

```
iw dev wlan0 set bitrates mcs-5 4
```

```
iw dev wlan0 set bitrates mcs-2.4 10
```

清除所有 tx bitrates 并将所有设置恢复正常：

```
iw dev wlan0 set bitrates mcs-2.4
iw dev wlan0 set bitrates mcs-5
```



## 设置 TX 功率 （tx power）

可以通过设备接口名或对应的 phy 名来设置 txpower

```
iw dev <devname> set txpower <auto|fixed|limit> [<tx power in mBm>]
iw phy <phyname> set txpower <auto|fixed|limit> [<tx power in mBm>]
```

需要注意，这里的参数单位是 毫贝-毫瓦（mBm） 而非常用的 分贝-毫瓦（dBm）。 \<mBm 功率\> = 100* \<dBm 功率\> 



## 节能

若要默认启用[节能](https://wireless.wiki.kernel.org/en/developers/documentation/ieee80211/power-savings)模式，可以用下面的命令：

```
sudo iw dev wlan0 set power_save on
```

对 mac80211 驱动而言，这意味着启动 [动态节能(Dynamic Power Save)](https://wireless.wiki.kernel.org/en/users/documentation/dynamic-power-save)

下面的命令可用于查询当前的节能设置：

```
iw dev wlan0 get power_save
```



## 使用 iw 添加新接口

iw 支持的模式如下：

- monitor
- managed [或 station]
- wds
- mesh [或 ap]
- ibss [或 adhoc]

关于模式的详细信息请阅读 [模式](https://wireless.wiki.kernel.org/en/users/documentation/modes) 文档。

用添加一个 monitor 接口举例：

```
iw phy phy0 interface add moni0 type monitor
```

上面的 `monitor` 是要添加的接口模式名，`moni0` 是自定义的接口名称，而 `phy0` 是硬件的 PHY 名称（通常来说都是 phy0，除非热插拔或重新加载其他模块）。如果 udev 配置错误，则新创建的虚拟接口可能会被它立即重新命名。可以使用 `ip link` 查看所有的接口。

要配置一个新的 managed 模式接口，可以用：

```
iw phy phy0 interface add wlan10 type managed
```

注意，当使用 hostapd 时，接口会被自动设置为 AP 模式

### 修改 monitor 接口标志

可以通过修改标志的方法来自定义创建的 monitor 接口，这当在用户系统上进行调试时非常有用。比如，假设你想帮一个用户解决问题，因为 mac80211 的 monitor 接口使用 radiotap 来向用户空间（userspace）传输额外的数据，我们可以利用这一点。也就是说，我们可以通过将接口设置为 full monitor 接口的方式，赖在不影响设备性能的情况下帮助用户"fish out data"。不设置额外标志的 monitor 接口的创建方式如下：

```
iw dev wlan0 interface add fish0 type monitor flags none
```

然后可以让用户在一个会话中使用 tcpdump：

```
tcpdump -i fish0 -s 65000 -p -U -w /tmp/fishing.dump
```

这些不同的 monitor 接口可以让你进一步扩展 radiotap，甚至可以使用 [厂商扩展](http://www.radiotap.org/Discussion/Vendor-Extensions) 来向 radiotap 中添加更多数据，以帮助调试设备的某些特性。

然而需要注意，这需要驱动严格遵守 mac80211 的标志请求，所以像 ath5k 和 ath9k 这样仍然基于运行模式控制标志的驱动需要在进行修改后才能利用本特性。

### 可选的 monitor 标志

- none
- fcsfail
- plcpfail
- control
- otherbss
- cook
- active



## 使用 iw 删除接口

```
iw dev moni0 del
```



## 虚拟 vif 支持

有一个单独的文档讨论该问题 [iw vif](https://wireless.wiki.kernel.org/en/users/documentation/iw/vif)



## 更新监管域（regulatory domain）

```
iw reg set alpha2
```

其中的 "alpha2" 是 [ISO/IEC 3166 alpha2](http://en.wikipedia.org/wiki/ISO_3166-1_alpha-2) 国家地区码。该信息用于设置 [监管基础设施（regulatory infrastructure）](https://wireless.wiki.kernel.org/en/developers/regulatory)

现在也可以用最新的 wpa_supplicant (0.6.7) 来修改该设置。例如只需要在配置文件中加一个 `country=US` 条目即可



## 使用 iw 创建和检查 Mesh Point

可以对支持 Mesh Point 操作的驱动添加 mesh 接口。Mesh Point 接口有一个 mesh_id 参数，可以被设置为 32 字节长。下面的命令使用 mesh_id "mymesh" 在 phy0 设备上创建了一个 "mesh0" 接口

```
iw phy phy0 interface add mesh0 type mp mesh_id mymesh
```

Mesh Point 接口默认被配置在 Channel 1 上。Mesh Point 会在接口 up 的时候开始运行。在默认配置中，Mesh Point 接口会自动探测和尝试与有着相同 mesh ID 的其他 Mesh Points 创建 Peer Links。

可以使用 station link 和  station statistics 查看 peer 列表和 Peer Link 状态

在发送流量后（比如 ping 另一个 mesh 节点），可以用下面的命令看 Mesh Paths 的列表：

```
iw dev mesh0 mpath dump
```

有关 Mesh Point 相关的命令和输出，请参阅 [HOWTO · o11s/open80211s Wiki (github.com)](https://github.com/o11s/open80211s/wiki/HOWTO)

iw 也提供了一些用于高级 Mesh Point 配置的命令，详细请参阅 [HOWTO · o11s/open80211s Wiki (github.com)](https://github.com/o11s/open80211s/wiki/HOWTO) 中的 Advanced Tinkering 一节



## 设置 WDS peer

WDS 模式是一个 IEEE 802.11 标准的非标准扩展，用于在站点上实现透明以太网桥接和无线客户端在不同 AP 之间的无缝交接漫游。因为这并不是标准内的东西，因此 WDS 在不同的驱动和厂商固件中的实现常常很不一样，导致 WDS 之间的不兼容。想要使用 WDS，应该在所有的无线设备上都部署相同的软硬件以维护兼容性。

创建 WDS peer ，要先创建 WDS 类型的接口，然后再设置 peer：

```
iw phy phy0 interface add wds0 type wds
iw dev wds0 set peer <MAC address>
```

驱动必须实现 cfg80211 的回调函数 `set_wds_peer()` ，wds 才能运行起来。mac80211 实现了这个回调函数，所以对应的 mac80211 驱动就应该能支持 WDS 类型的接口。WDS 在 TX 帧时，需要做的第一件事是使用 peer 的地址替换掉 802.11 头中的第一个地址。如果你能决定 AP 以及其他的 客户端 / peer 所使用的软件，那么在使用 WDS 之前，最好先考虑一下使用下面介绍的 4 地址模式（4-address mode）。



## 在 AP 和客户端使用 4地址模式（ 4-address ）

> 4 地址模式是一个允许在 wifi 和以太网之间实现无线桥接的模式。在该模式中，每个无线帧都有 4 个 MAC 地址：原始数据包的源地址和目的地址、当前无线链路上的发送者与接收者的地址。如此一来，无线设备就可以在不改变原始数据包 MAC 地址的情况下、实现在不同的网络间扮演网桥的角色
>
> 正在传递数据的 802.11 设备可能并不是当前 L2 流量的实际发送者与接收者，因此如果还想要体现实际的发送与接收者，数据包中就需要包含四个地址：
>
> - 发送者地址（Transmitter Address, TA）
> - 接收者地址（Receiver Address, RA）
> - 源地址（Source Address, SA）
> - 目的地地址（Destination Address, DA）

在一些场景中需要运行由一个 AP 和多个客户端组成的网络，且要将这些客户端桥接到同一个网络中。为实现该目标，客户端和 AP 都需要传输 4 地址的帧，该帧包含源、目的地 MAC 地址。OpenWrt 就是使用 4地址模式为 mac80211 驱动提供 WDS 模式支持的，也就是说如果你在你的 [OpenWrt 无线配置](http://wiki.openwrt.org/doc/uci/wireless)中启用了 wds 选项，那么实际上就是启用了 4 地址模式。4 地址模式和其他的 WDS 实现并不兼容，也就是说你需要所有的 endpoint 都使用 4 地址模式以确保 WDS 正常工作。

Linux 无线为 AP 和 STA 提供了 4 地址支持，但每个驱动都需要显式定义对该模式的支持。只要能够以 AP 或 STA 模式运行，则所有的 mac80211 驱动支持 4 地址模式

在 AP 端，可以通过将每个客户端都各自划分进单独的、配置为 4 地址模式的 AP VLANs 的方式，只对这特定的每一个客户端启用 4 地址模式。这样的一个 AP VLAN 就是为这一个客户端单独存在的，因此该客户端会被配置为这个 AP VLAN 接口中所有流量的目的地，而无视数据包头中的目的地 MAC 地址。相较于通常的 WDS 模式而言，该模式的优点是更加易于配置并且不需要在任何一端维护一个静态的 peer MAC 地址列表。4 地址模式与 WDS 模式并不兼容。

可以用 `4addr on` 创建一个启用 4 地址模式的接口，例如：

```
iw phy phy0 interface add sta0 type managed 4addr on
```

在该模式下，若新接口在网桥中，则可以在 wpa_supplicant 中使用 `-b` 选项，来让接口在网桥上监听 EAPOL。

在 hostapd 中，可以通过在 hostapd.conf 中添加标志的方式实现同样的目标：

```
wds_sta=1
```



## 创建数据包合并规则（packet coalesce rules）

在绝大多数情况下，收到 IPv4/IPv6 的多播/广播数据包的 host 都不会对包做任何处理。这些实际上并不被需要的广播传输就造成了无谓的计算与能量消耗。

包合并特性能够通过在固件/硬件中先把包缓存一段时间的方式，帮助减少 host 接收中断（receive interrupts）的次数。当发生下面的事件时，才会触发接收中断：

- 硬件计时器超时。该超时时间是由相应的合并规则预先配置的。
- 硬件的合并缓冲区满
- 数据包不匹配任何预先配置的合并规则

可以用 `iw phy0 info` 这样的命令来查看对于合并配置的支持信息，相应的样例输出如下：

```
Coalesce support:
         * Maximum 8 coalesce rules supported
         * Each rule contains upto 4 patterns of 1-4 bytes,
           maximum packet offset 50 bytes
         * Maximum supported coalescing delay 100 msecs
```

要创建合并规则，需要配置下列参数：

- 最大合并延迟
- 用于匹配数据包的模式（ pattern ）列表
- 合并条件。模式 'match' 或 'no match'

可以在一个配置文件中包含多个这样的规则。例如，要使用 `coalesce.conf` 文件启用合并规则，可以用下面的命令：

```
iw phy phy0 enable coalesce.conf
```

 `coalesce.conf` 文件内容如下：

```
delay=25
condition=0
patterns=8+34:xx:ad:22,10+23:45:67,59:33:xx:25,ff:ff:ff:ff
delay=40
condition=1
patterns=12+00:xx:12,23:45:67,46:61:xx:50
```

如果要查看当前的合并配置，可以用 `iw phy phy0 coalesce show`：

```
$ iw phy phy0 coalesce show
Coalesce is enabled:
Rule - max coalescing delay: 25msec condition:match
 * packet offset: 8 pattern: 34:--:ad:22
 * packet offset: 10 pattern: 23:45:67
 * packet offset: 0 pattern: 59:33:--:25
 * packet offset: 0 pattern: ff:ff:ff:ff
Rule - max coalescing delay: 40msec condition:not match
 * packet offset: 12 pattern: 00:--:12
 * packet offset: 0 pattern: 23:45:67
 * packet offset: 0 pattern: 46:61:--:50
```

下面的命令可以关闭合并特性

```
iw phy phy0 coalesce disable
```

当没有配置合并特性时，输出如下

```
$ iw phy phy0 coalesce show
Coalesce is disabled.
```

