# nftables的协议簇（Nftables families）



Netfilter允许在多个网络级别上进行过滤。相对于iptables，每个级别都有一个单独的工具:iptables, ip6tables, arptables, ebtables。使用nftables，多个网络级别被抽象成**families**，所有这些级别都由单一工具`nft`提供服务。

请注意，您看到的流量/数据包（traffic/packets）以及在网络堆栈中的位置取决于您正在使用的[**hook**](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks)（钩子）。

## ip

这个族的表参见 [IPv4](https://en.wikipedia.org/wiki/IPv4)流量/报文。`iptables`工具是遗留的 x_tables 等价工具。

## ip6

 [IPv6](https://en.wikipedia.org/wiki/IPv6)流量/报文族表。ip6tables工具是遗留的x_tables等价工具。

## inet

这个家族的表可以看到IPv4和IPv6流量/数据包，简化了双栈支持。

在inet协议簇的一个表中，IPv4和IPv6数据包遍历相同的规则。IPv4数据包规则不影响IPv6数据包。这两个三层协议（L3 protocols）的规则都影响这两个协议。

例子：

```
# 该规则只影响IPv4数据包:（往inet协议簇的filter表input链添加规则）
nft add rule inet filter input ip saddr 1.1.1.1 counter accept

# 该规则只影响IPv6数据包:
nft add rule inet filter input ip6 daddr fe00::2 counter accept

# 这些规则同时影响IPv4和IPv6数据包:
nft add rule inet filter input ct state established,related counter accept
nft add rule inet filter input udp dport 53 accept
```

在nftables 0.9.7和Linux内核4.10中新增的是inet协议簇的 [*ingress*](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks)钩子，它在与netdev *ingress*钩子相同的位置进行过滤。

## arp

这个协议簇的表在内核完成任何三层处理之前看到arp级(即二层)流量。arptables工具是遗留的 x_tables 等价工具。

## bridge

这类表表示通过[bridges](https://wiki.linuxfoundation.org/networking/bridge)(即交换)的流量/包。没有关于三层协议的假设。

ebtables工具是遗留的x_tables等价工具。一些旧的x_tables模块(如*physdev*)最终也将从*bridge*协议簇中提供服务。

请注意，*bridge*协议簇没有nf_conntrack集成。

## netdev

*netdev* 协议簇与其他协议簇不同之处在于，它用于创建绑定单个网络接口的基链。这样的基链检查指定接口上的所有网络流量，不考虑二层（L2）或三层(L3)协议。因此，您可以从这里过滤ARP流量。遗留的x_tables中没有与*netdev* 协议簇等价的。

这个协议簇的主要(或者说唯一？)用途是用于[*ingress* hook](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks)的基链，这是Linux kernel 4.2中的新功能。*ingress* 链在网卡驱动（NIC driver）将网络数据包上传到网络堆栈之后才看到它们。这个位置在数据包路径中非常早期的，非常适合丢弃 [DDoS](https://en.wikipedia.org/wiki/Denial-of-service_attack#Distributed_attack)攻击相关的数据包。从*ingress*类型的链中丢弃数据包的效率是从*prerouting*类型链的两倍。(注意，在*ingres*链中，分片成功的数据报（ fragmented datagrams）尚未被重新组装。所以，匹配规则`ip saddr`和``daddr`所有的ip数据包均生效，但匹配四层协议头部（L4 headers）如`udp dport`只生效为未分片的数据包，或第一个片段。）

 *ingress*钩子提供了一个替代`tc`入接口过滤的方法。您仍然需要tc来进行流量整形/队列管理。

你也可以使用*ingress*钩子来实现[负载平衡](https://wiki.nftables.org/wiki-nftables/index.php/Load_balancing)，包括直接服务器返回(DSR)，[据报道它的速度要快10倍](https://netdevconf.org/1.2/slides/oct6/08_nftables_Load_Balancing_with_nftables_II_Slides.pdf)。