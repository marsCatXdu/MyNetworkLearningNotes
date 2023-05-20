> 本文基于以下内容翻译、删改而成：
>
> [Becoming friends with NetworkManager | Enable Sysadmin (redhat.com)](https://www.redhat.com/sysadmin/becoming-friends-networkmanager)
>
> [NetworkManager - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/NetworkManager)

NetworkManager 是一个系统网络服务，其能够管理网络设备以及它们到网络的连接。NetworkManager 能够管理以太网、wifi、移动宽带（WWAN, mobile broadband）和 PPPoE 设备等。

简单来说，其为这些功能提供支持：

- WiFi 连接
- WAN 连接（和 ModemManager 共同提供）
- Ethernet 连接
- 创建 WiFi AP
- 共享连接
- VPN 连接

NetworkManager 配置中的两大基本概念是 **设备(device)** 和 **连接(connection)** 。NetworkManager 对每个设备都会保存一些与之相关的信息，包括：

- 该设备是否由 NetworkManager 管理
- 该设备的可用连接
- 该设备上已经激活的连接（如果有）

"连接"代表应用到设备上的配置，一个连接实际上是多个属性组成的一个列表。一组相同方面的属性被归类为一组设置（比如 ipv4 设置组（setting group）就包含地址、网关、路由等属性）。

nmcli 是对 NetworkManager 进行配置、管理以及使用的命令行工具。下面将介绍如何通过 nmcli 使用 NetworkManager

## nmcli 基础

### 设备

列出 NetworkManager 检测到的设备：

```
$ nmcli device
DEVICE   TYPE       STATE           CONNECTION
enp1s0   ethernet   connected       ether-enp1s0
enp7s0   ethernet   disconnected    --
enp8s0   ethernet   disconnected    -- 
```

在这个例子中，NetworkManager 在系统中检测到了三个以太网设备，只有第一个 `enp1s0` 有一个已经激活的连接（也就是说这个设备已经被配置(configured)了）。

如果想让 NetworkManager 暂时停止对某个设备的管理，可以临时 *unmanage* 这个设备：

```
$ nmcli device set enp8s0 managed no
$ nmcli device
DEVICE   TYPE       STATE          CONNECTION
enp1s0   ethernet   connected      ether-ens3
enp7s0   ethernet   disconnected   --
enp8s0   ethernet   unmanaged      -- 
```

这是个临时方案，重启后就会被重置。

如果要再启动，使用 `nmcli device set enp8s0 managed yes` 即可

如果直接使用 `nmcli` 而不加任何参数，则将打印每个设备的 IP 配置情况

```
$ nmcli
enp1s0: connected to enp1s0
      "Red Hat Virtio"
      ethernet (virtio_net), 52:54:00:XX:XX:XX, hw, mtu 1500
      ip4 default
      inet4 192.168.122.225/24
      route4 0.0.0.0/0
      route4 192.168.122.0/24
      inet6 fe80::4923:6a4f:da44:6a1c/64
      route6 fe80::/64
      route6 ff00::/8

enp7s0: disconnected
      "Intel 82574L"
      ethernet (e1000e), 52:54:00:XX:XX:XX, hw, mtu 1500

enp8s0: unmanaged
      "Red Hat Virtio"	
      ethernet (virtio_net), 52:54:00:XX:XX:XX, hw, mtu 1500
```

### 连接

列出所有可用连接：

```
$ nmcli connection
NAME                 UUID                                  TYPE       DEVICE
ether-enp1s0         23e0d89e-f56c-3617-adf2-841e39a85ab4  ethernet   enp1s0
Wired connection 1   fceb885b-b510-387a-b572-d9172729cf18  ethernet   --
Wired connection 2   074fd16d-daa6-3b6a-b092-2baf0a8b91b9  ethernet   --
```

例子中的输出说明，当前只有 `ether-enp1s0` 这一个连接处于激活状态，且被应用于 enp1s0 设备上。另外两个连接则并没有被激活。

关闭连接：

```
$ nmcli connection down ether-enp1s0
```

再次启动连接：

```
$ nmcli connection up ether-enp1s0
```

查看某个连接的详细属性信息：

```
$ nmcli connection show ether-enp1s0
connection.id:                     ether-enp1s0
connection.uuid:                   23e0d89e-f56c-3617-adf2-841e39a85ab4
connection.stable-id:              --
connection.type:                   802-3-ethernet
connection.interface-name:         enp1s0
connection.autoconnect:            yes
connection.autoconnect-priority:   -999
connection.autoconnect-retries:    -1 (default)
connection.auth-retries:           -1
connection.timestamp:              1559320203
connection.read-only:              no
[...]
```

上面这些属性的含义以及别名对照如下

| 属性                      | 描述                                                         | 别名        |
| ------------------------- | ------------------------------------------------------------ | ----------- |
| connection.id             | 人类可读的连接名称                                           | con-name    |
| connection.uuid           | 用于唯一识别连接的 UUID                                      | 无          |
| connection.type           | 连接类型                                                     | type        |
| connection.interface-name | 绑定该连接的特定设备。该连接只能在这个设备上激活             | ifname      |
| connection.autoconnect    | 是否自动激活该连接                                           | autoconnect |
| ipv4.method               | 连接的 IPv4 方法：auto, disabled, link local, manual, shared | 无          |
| ipv4.address              | 连接的静态 IPv4 地址                                         | 无          |

当 autoconnect 启用（`=yes`，默认启用）时，只要 `interface-name` 设备就绪且其上没有已经激活的连接，该连接就会被 NetworkManager 自动激活。如果 `ipv4.method` 被设置为 `auto` ，则会从 DHCP 获取 IPv4 配置。

如果想要手动设置静态配置，可以将 `ipv4.method` 设置为 `manual`，然后在 `ipv4.addresses` 属性中指定静态 IP 地址和子网（ 用 CIDR 写法 ）。相关属性的详细的描述和含义请查阅 man （`man nm-settings`）

如果想要修改一个连接的属性，可以用 `nmcli connection modify` 命令。下面演示将一个连接改为静态 IPv4 地址 10.10.10.1， 网关 10.10.10.254 ， DNS 10.10.10.254

```
$ nmcli connection modify ether-enp1s0 ipv4.method manual \
ipv4.address 10.10.10.1/24 \
ipv4.gateway 10.10.10.254 ipv4.dns 10.10.10.254
```

该命令会永久性修改该连接。新的设置需要在下次使用设备激活连接时才会生效。可以手动重新激活连接以使新设置生效：

```
$ nmcli connection up ether-enp1s0
```

可以像下面这样取消 NetworkManager 自动激活该连接

```
$ nmcli connection modify ether-ens1s0 connection.autoconnect no
```

这之后就需要手动控制连接了

## 使用 nmcli 创建新连接

通过 nmcli 的子命令 add 来新增连接（下面的 con 是 connection 的简写）

```
$ nmcli con add type ethernet ifname enp0s1 con-name enp0s1_dhcp autoconnect no
$ nmcli con
NAME UUID TYPE DEVICE
ether-enp1s0 23e0d89e-f56c-3617-adf2-841e39a85ab4 ethernet enp1s0
enp0s1_dhcp 64b499cb-429f-4e75-a54d-b3fd980c39aa ethernet --
Wired connection 1 fceb885b-b510-387a-b572-d9172729cf18 ethernet --
Wired connection 2 074fd16d-daa6-3b6a-b092-2baf0a8b91b9 ethernet --
```

因为并没有指定任何 IPv4 属性，因此 `ipv4.method` 属性值为默认的 `auto` ，该连接会自动从 DHCP 服务器获取 IPv4 配置。

在创建连接时，根据连接类型 `connection.type` 的不同（`ethernet`, `wifi`, `bond`, `vpn` 等），会有不同的必须包含的属性要求，如果没有提供相应的必选属性则 `nmcli` 会报错并给出相应缺失属性的名称。

也可以添加 `--ask` 选项，使用交互式的方式来创建连接：

```
$ nmcli --ask con add
Connection type: ethernet
Interface name [*]: enp0s1
There are 3 optional settings for Wired Ethernet.
Do you want to provide them? (yes/no) [yes] no
There are 2 optional settings for IPv4 protocol.
Do you want to provide them? (yes/no) [yes] no
There are 2 optional settings for IPv6 protocol.
Do you want to provide them? (yes/no) [yes] no
There are 4 optional settings for Proxy.
Do you want to provide them? (yes/no) [yes] no
Connection 'ethernet-enp0s1' (64b499cb-429f-4e75-a54d-b3fd980c39aa) successfully added.
```

此外，也可以用 `nmcli con edit` 来使用一个 nmcli 编辑器创建连接

## nmcli 命令 cheat sheet

下面整理了 `nmcli connection` 的相关命令

| 命令   | 参数                         | 描述                                             |
| ------ | ---------------------------- | ------------------------------------------------ |
| down   | *连接名*                     | 关闭指定链接                                     |
| up     | *连接名*                     | 启动指定链接                                     |
| show   | [连接名]                     | 不加参数时列出所有连接。加名称或 UUID 可指定连接 |
| modify | *连接名 {属性名, 属性值}...* | 修改连接的熟悉                                   |
| add    | *连接名 {属性名, 属性值}...* | 使用给定选项创建新连接                           |



## 一些 nmcli 命令例子

列出附近的 Wi-Fi 网络:

```
$ nmcli device wifi list
```

连接到 Wi-Fi 网络:

```
$ nmcli device wifi connect SSID_or_BSSID password password
```

连接到隐藏的 Wi-Fi 网络:

```
$ nmcli device wifi connect SSID_or_BSSID password password hidden yes
```

在 `wlan1` 网络接口上连接 Wi-Fi 网络:

```
$ nmcli device wifi connect SSID_or_BSSID password password ifname wlan1 profile_name
```

断开特定接口的连接:

```
$ nmcli device disconnect ifname eth0
```

获取连接信息列表，包括连接名称、UUID、类型以及设备:

```
$ nmcli connection show
```

激活连接 (即使用已经有的配置连接网络):

```
$ nmcli connection up name_or_uuid
```

删除一个连接:

```
$ nmcli connection delete name_or_uuid
```

列出当前网络设备以及对应状态:

```
$ nmcli device
```

关闭 Wi-Fi:

```
$ nmcli radio wifi off
```

