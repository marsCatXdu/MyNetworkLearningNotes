# Linux network namespaces （网络命名空间）

> 本文主要内容翻译、删改自 [Introducing Linux Network Namespaces](https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/) ，原作者 Scott
>
> 参考补充文章：[Deep Dive into Linux Networking and Docker - Bridge, vETH and IPTables (arriqaaq.com)](https://aly.arriqaaq.com/linux-networking-bridge-iptables-and-docker/)

Linux Namespace 是 Linux 系统中的一个资源抽象方式，能够实现资源隔离。目前有七种 namespace ：Cgroup, IPC, Network, Mount, IPD, User, UTS。本文讲的是 Network Namespace

Network Namespace 能够实现网络相关资源的分割及隔离。运行在一个单独 network namespace 中的进程拥有它自己的网络设备、路由表、防火墙规则等。

通常来说，一个 Linux 系统会在系统层面上，共享同一组网络接口和路由表条目，用户可以通过策略路由（policy routing）来修改路由表的条目（相关阅读：[A Quick Introduction to Linux Policy Routing](https://blog.scottlowe.org/2013/05/29/a-quick-introduction-to-linux-policy-routing/)、[A Use Case for Policy Routing with KVM and Open vSwitch](https://blog.scottlowe.org/2013/05/30/a-use-case-for-policy-routing-with-kvm-and-open-vswitch/)）。但借助 Network Namespaces ，我们便可以创建多个网络接口和路由表实例，且让它们相互独立。

## 创建及列出 Linux Network Namespaces

> 所有的命令都应以 root 权限执行

创建：

```
# ip netns add <新的 namespace 名称>
```

比如要创建一个 "blue"：

```
# ip netns add blue
```

使用 list 即可查看目前的 namespace

```
# ip netns list
```

## 为 Network Namespace 指定接口（ interface ）

在创建了 namespace 后，需要为该 namespace 分配及配置接口，以实现所需的网络连接。下面用一个虚拟以太网接口（Virtual Etherenet, veth）举例。veth 接口总是成对出现，像管道一样相连 —— 进入一端的东西一定会从另一端出现。因此，我们可以使用 veth 接口，通过物理接口所在的 "default" 或 "global" namespace 来实现自定义 namespace 与外界的联通。

> veth 是本地以太网隧道（tunnel），veth 设备成对出现，发往一个设备的数据包能够立即在另一个设备中收到。
>
> veth 就和网线一样，一端连着 host 网络，另一端连着我们创建的 namespace

示例如下

首先，创建一个 veth 对（下面以 veth0、veth1 为例，这两个名字可以自定义）：

```
# ip link add veth0 type veth peer name veth1
```

这样可以一次创建出 veth0，veth1 两个接口，并把它们连在一起。创建完成后可以用 `ip link list` 查看，输出参考如下

```
3: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ea:52:58:cd:d7:07 brd ff:ff:ff:ff:ff:ff
4: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 3a:1b:82:2a:d4:d9 brd ff:ff:ff:ff:ff:ff
```

此时，这两个 veth 都和物理接口一样，属于 "default" 或 "global" namespace

接下来，用下面的命令可以，借助 veth 将最开始创建的 blue namespace 连接到 global namespace 上：

```
# ip link set veth1 netns blue
```

此时再跑一下 `ip link list` ，会发现 veth1 消失了，veth0 后面的字样也有所改变

```
4: veth0@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 3a:1b:82:2a:d4:d9 brd ff:ff:ff:ff:ff:ff link-netns blue
```

这时的 veth1 已经属于 blue namespace 了，使用下面的命令来查看 blue namespace

```
# ip netns exec blue ip link list
```

> 这句命令的含义：`ip netns exec blue` 说明接下来的命令是要在 blue namespace 中执行的，后面接着的 `ip link list` 就是实际被在 blue namespace 中执行的命令

输出参考如下。这里的 veth1 就是前面与 veth0 成对创建的那个 veth 接口

```
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: veth1@if4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ea:52:58:cd:d7:07 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

## 配置 Network Namespace 中的接口

现在我们来配置 blue 的 veth1 接口

```
# ip netns exec blue ip addr add 10.1.1.1/16 dev veth1
# ip netns exec blue ip link set dev veth1 up
```

第一条命令，用 `ip addr` 为 veth1 接口分配了一个 IP 地址。第二条命令用 `ip link` 启动了 veth1 接口

veth1 接口启动后，我们就可以用下面几个命令来确认 veth1 的确属于一个完全不同的 namespace，其具有与 global 相互独立的网络配置

- 分别执行 `ip addr list` 和 `ip netns exec blue ip addr list` ，会发现接口在不同的 namespace 中有着不同的 IP 地址
- 分别执行 `ip route list` 和 `ip netns exec blue ip route list` ，会发现每个 namespace 有着不同的路由表条目以及不同的默认网关

例如：

```
[lijingwei@localhost ~]$ ip addr list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    (略)
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 4e:19:26:6d:0f:09 brd ff:ff:ff:ff:ff:ff
    inet 192.168.64.10/24 brd 192.168.64.255 scope global dynamic noprefixroute enp0s1
       valid_lft 83809sec preferred_lft 83809sec
    inet6 fd4c:1c05:d22d:703d:4c19:26ff:fe6d:f09/64 scope global dynamic noprefixroute 
       valid_lft 2591948sec preferred_lft 604748sec
    inet6 fe80::4c19:26ff:fe6d:f09/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
4: veth0@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 3a:1b:82:2a:d4:d9 brd ff:ff:ff:ff:ff:ff link-netns blue
[lijingwei@localhost ~]$ sudo ip netns exec blue ip addr list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: veth1@if4: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether ea:52:58:cd:d7:07 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.1.1.1/16 scope global veth1
       valid_lft forever preferred_lft forever
```

```
[lijingwei@localhost ~]$ ip route list
default via 192.168.64.1 dev enp0s1 proto dhcp src 192.168.64.10 metric 100 
192.168.64.0/24 dev enp0s1 proto kernel scope link src 192.168.64.10 metric 100 
[lijingwei@localhost ~]$ sudo ip netns exec blue ip route list
10.1.1.0/16 dev veth1 proto kernel scope link src 10.1.1.1 linkdown
```

##  将 Network Namespace 连接到物理网络

network namespace 与物理网络的连接需要通过网桥 bridge （比如 Open vSwitch bridge 或 linux bridge）来进行，只要将一或多个物理接口和一个 veth 接口连入网桥即可。也可根据需要，将不同的 namespace 连入不同的物理网络或物理网络上的不同 VLAN。

网桥是一个能够连接多个网络或网段，让他们看起来就像是处于同一网络的设备。

网桥可以将两个以太网段以协议无关的方式连接起来，其中的数据包基于以太网地址转发而非 IP 地址（根据 IP 地址转发是路由器的工作）—— 网桥是一个 Layer 2 设备，其存在对上层协议来说是无感的。路由让多个相互独立的网络实现通信，网桥则让这些网络看起来就像是同一个网络

比如 Docker 在启动时就会在 host 上创建一个 `docker0` Linux 网桥，不同容器的 interface 都和网桥通信，网桥再将流量"代理"到外部。同一个 host 上的多个容器也可以通过网桥实现相互通信。下面我们创建一个网桥，然后将外面的 veth 连接到网桥上：

```
ip addr add 10.1.1.2/16 dev veth0    # 分配一个同网段的 IP
ip link add br0 type bridge          # 创建网桥 br0
ip link set br0 up                   # 启动网桥 br0
ip link set veth0 master br0         # 将 veth 这一端连接到 br0
ip addr add 10.1.1.3/16 dev br0      # 给 br0 分配一个 IP
```

这么一来，外部的网络接口就连到网桥了。现在 blue namespace 的路由表还只有自己 IP 子网范围内的路由条目。因为 veth 已经接到网桥上了，桥接网络的地址就可以被 blue 获取到了。下面为 blue 添加路由

```
ip netns exec blue ip route add default via 10.1.1.3
```

然后我们自定义 namespace 的网络，就能通过网桥和外部连通了。

## 在 namespace 中运行进程

```
ip netns exec bluw python3 -m http.server 8000
```









