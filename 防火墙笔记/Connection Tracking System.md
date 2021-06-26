# 连接跟踪系统（Connection Tracking System）

nftables使用netfilter的连接跟踪系统(通常称为*conntrack* 或 *ct* )将网络数据包(network packets)与连接以及这些连接的状态关联起来。nftables规则集（ruleset）通过应用基于数据包是否是被跟踪连接的有效部分的策略来执行有状态防火墙（ [stateful firewalling](https://en.wikipedia.org/wiki/Stateful_firewall)）。

## [Conntrack sysfs variables](https://git.kernel.org/pub/scm/linux/kernel/git/pablo/nf-next.git/tree/Documentation/networking/nf_conntrack-sysctl.rst)

### References

| 参考文档                                                     | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [*Netfilter's Connection Tracking System*, Pablo Neira Ayuso, ;login: Vol. 31 No. 3, 2006](http://people.netfilter.org/pablo/docs/login.pdf) | 连接跟踪设计和实现细节。                                     |
| [*Netfilter Connection Tracking and NAT Implementation*, Magnus Boye, Aalto University School of Electrical Engineering, 2012](https://wiki.aalto.fi/download/attachments/69901948/netfilter-paper.pdf) | 连接跟踪内部的更多细节。还深入研究了netfilter NAT及其一些潜在的漏洞。 |
| [conntrack-tools documentation](http://conntrack-tools.netfilter.org/support.html) | *conntrack*命令行工具允许您检查和维护当前跟踪的连接。<br>*conntrackd* 服务为附加的七层（L7）协议添加对用户空间 *tracking helpers* 的支持, 包括 DHCPv6, MDNS, SLP, SSDP, RPC, NFSv3, and Oracle TNS.<br> **注意**：不幸的是，这里的在线文档似乎没有保持最新。请参阅下面的Debian手册页以获得更多的最新文档。 |
| [Debian man conntrack](https://manpages.debian.org/unstable/conntrack/conntrack.8.en.html)[Debian man conntrackd](https://manpages.debian.org/unstable/conntrackd/conntrackd.8.en.html) | 最近的连接跟踪工具手册页，由Debian提供。                     |
| [*The state machine*, Ch. 7 of Oskar Andreasson's *Iptables Tutorial*](https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#STATEMACHINE) | 详细介绍conntrack，尽管使用的是旧的iptables和/proc/net/ip_conntrack(现在分别被nftables和conntrack命令取代). |

