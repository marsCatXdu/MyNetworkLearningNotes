[iptables - ArchWiki (archlinux.org)](https://wiki.archlinux.org/title/iptables)

[The Beginner’s Guide to iptables, the Linux Firewall (howtogeek.com)](https://www.howtogeek.com/177621/the-beginners-guide-to-iptables-the-linux-firewall/)

[An introduction to Linux bridging commands and features | Red Hat Developer](https://developers.redhat.com/articles/2022/04/06/introduction-linux-bridging-commands-and-features#)

iptables 是一个用于配置 Linux 内核防火墙的命令行工具，其属于 Netfilter 项目。术语 `iptables` 也被用于直接指代内核级别的防火墙。iptables 可以直接使用命令行工具进行配置，也可以使用多种图形化的前端工具配置。iptables 专用于 IPv4，IPv6 则使用专门的工具 ip6tables。这两个工具的语法相同，在具体选项上有所区别。

> 提示：iptables 是一个经典框架，还有一个叫做 nftables 的更现代的工具，其自己构建了一个到 iptables 的兼容层。

## 基本概念

iptables 被用于检查（inspect）、修改、转发、重定向以及丢弃（drop） IP 数据包。内核已经包含了用于过滤 IP 数据包的代码，这些过滤功能被组织为了一系列的 tables，每个 table 都面向一类特定的场景。

这些 tables 组成一组预定义好的 chains ，chains 包含一系列会被按顺序遍历的 rules 。每条 rule 包含一个用于进行匹配的条件 (predicate) 和一条对应的动作（action，被称为 "目标 target"），当条件值为真（即条件成立）时就执行该动作，如果不满足条件就接着判断下一条 rule。

如果 IP 数据包跑完了所有的内置 chain（包括空 chain），则 chain 的目标就会决定该 IP 数据包的最终去向。iptables 就是让用户管理这些 chains/rules 的工具。IP 路由对绝大多数新用户而言看起来有些复杂，但其实在常见场景（NAT 或基本的互联网防火墙）中也不至于太复杂

下图是理解 iptables 的关键。图中每个椭圆中上半部分的小写字母代表 table ，下面的大写字母代表 chain。

![img]([TBD]iptables.assets/tables_traverse.jpg)

从任何一个网络接口上进来的每一个 IP 数据包都要按照这个图从上到下完整跑一遍。注意这里说的是【每一个网络接口】，这里有个常见的误解——从内部接口进来的数据包，和从连接到互联网的外部接口进来的数据包，有着不同的处理流程——事实并非如此，所有接口数据包的处理流程都完全一致，但你可以手动配置成让他们用不同的处理流程。

有些包是要发给本地进程的，这些包会从最上面走下来，并停在左侧的 `<Local Process>` 这里。有些包是有本地进程发出的，这些包会 `<Local Process>` 出发，一直往下跑完这个图表的下半部分。关于这个表的详细解读请参阅[Iptables Tutorial 1.2.2 (frozentux.net)](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#TRAVERSINGOFTABLES)



### Tables

iptables 有五张表：

1. `raw` ：仅用于配置数据包（configuring packets），因此不对包进行连接追踪（connection tracking）
2. `filter`：默认表。防火墙一般在该表中关联对数据包的处理动作。其包含 **INPUT**（用于发往本地 socket 的数据包）、**FORWARD**（用于要通过本机被路由的数据包）和 **OUTPUT**（用于本地生成的数据包）
3. `nat`：用于网络地址转换（NAT，即端口转发(port forwarding) ）。
4. `mangle`：用于对特定数据包进行修改（specialized packet alterations）
5. `security`：用于强制访问控制（Mandatory Access Control, MAC）网络规则（例如 SELinux ）

在大多数场景中，只会用到 filter 和 nat 表。其他的表通常会在涉及到多路由、路由选择等复杂场景时才会用到。



### Chains

tables 包含 chains，chain 是一个由 rules 组成的有序列表。默认表 `filter` 包含三个内建的 chain：`INPUT`, `OUTPUT` 和 `FORWARD`，如上图所示，它们会在数据包过滤过程中的不同时刻被激活。

nat table 包含 `PREROUTING`, `POSTROUTING` 和 `OUTPUT` chains













