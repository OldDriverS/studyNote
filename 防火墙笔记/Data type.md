# Data types

## *nft describe*

您可以使用`nft describe`获取有关数据类型的信息，找出特定选择器的数据类型，并为该选择器列出预定义的符号常量。一些例子:

```
% nft describe iif
meta expression, datatype iface_index (network interface index) (basetype integer), 32 bits

% nft describe iifname
meta expression, datatype ifname (network interface name) (basetype string), 16 characters

% nft describe tcp flags
payload expression, datatype tcp_flag (TCP flag) (basetype bitmask, integer), 8 bits

pre-defined symbolic constants (in hexadecimal):
        fin                             0x01
        syn                             0x02
        rst                             0x04
        psh                             0x08
        ack                             0x10
        urg                             0x20
        ecn                             0x40
        cwr                             0x80
```

## List of data types



### 日期与时间类型

| 数据类型 | 描述                                                         | 表达式                                                       | 备注                                                         |
| :------: | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
|   day    | 接受数据包的星期 (8位整数, with pre-defined symbolic constants)：<br>*Sunday*<br/> *Monday<br/>* *Tuesday* <br/>*Wednesday* <br/>*Thursday<br/>* *Friday* <br/>*Saturday* | [*meta day*](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_metainformation) | *Sunday* = 0, *Saturday* = 6.<br/>符号常量不区分大小写, 接受独特的缩写: *Sun* = *sun* = *Sunday* = 0. |
|   hour   | 接收数据包的时间 (32位整数).指定为24小时格式的字符串, hh:mm[:ss]. | [*meta hour*](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_metainformation) | 秒是可选的: *17:00* = *17:00:00*.                            |
|   time   | 接收包的相对时间 (64位整数).                                 | [*meta time*](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_metainformation) | 可以指定为ISO格式的日期。<br/>例如， "2019-06-06 17:00"。小时和秒是可选的，如果需要可以省略。如果省略，将假定为午夜。<br/>下面三个是等价的： "2019-06-06" = "2019-06-06 00:00" = "2019-06-06 00:00:00"。当指定一个整数时，假定它是一个UNIX时间戳。 |



### Network interface types

|  数据类型   | 描述                                                         | 表达式                                                       | 备注                                                         |
| :---------: | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
|  devgroup   | Device group (32位整数).                                     | [*meta* {*iifgroup* \| *oifgroup*}](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_metainformation) | 可以用/etc/iproute2/group中数字或符号名称来指定.             |
| iface_index | 网络接口索引 (32位无符号整数).                               | [*meta* {*iif* \| *oif*}](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_metainformation) | 可以用数字或现有接口的名称指定。<br/>对于名称或索引可能被更改的网络接口（interface），请使用*ifname* (例如动态增加或删除的接口). |
| iface_type  | 网络接口类型 (16 bit integer, with pre-defined symbolic constants):*ether* *ppp* *ipip* *ipip6* *loopback* *sit* *ipgre* | [*meta* {*iiftype* \| *oiftype*}](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_metainformation) |                                                              |
|   ifkind    | 接口类名称 (16位字符串).                                     | [*meta* {*iifkind* \| *oifkind*}](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_metainformation) | dev->rtnl_link_ops->kindThe 。<br/>`man 8 ip-link` TYPES 部分列出了有效的**ifkinds**.它至少少了一个: *tun*。 |
|   ifname    | Interface name (16 byte string).                             | [*meta* {*iifname* \| *oifname*}](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_metainformation) | 不一定要存在。<br/>比*iface_index*慢，但对于可以动态加载的网络接口来说很好。 |



### Ethernet types

|  数据类型  | 描述                                                         | 表达式                                                       | 备注                                                         |
| :--------: | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| ether_addr | Ethernet address (48 bit integer).                           | ether {saddr \|daddr}<br>arp {saddr \|daddr} ether           |                                                              |
| ether_type | [EtherType](https://en.wikipedia.org/wiki/EtherType) (16位长度整数, 使用预定义的常量，值是确定的):*arp* *ip* *ip6* *vlan* | [*meta protocol*](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_metainformation) | [ether.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/if_ether.h) 列出已知类型。<br/>注意：**ether.h**按网络顺序列出EtherTypes，而**nft**在x86上使用little-endian顺序。(检查 `nft describe ether_type`的输出) |



### ARP types

|              |                                                              |                                                              |                                                              |
| :----------: | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **数据类型** | **描述**                                                     | **表达式**                                                   | **备注**                                                     |
|              | ARP HLEN, 以字节为单位的硬件地址长度 (8 bit integer)         | [*arp hlen* «HLEN»](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_headers#Matching_ARP_headers) | 无名称的8位长度的整数。<br/>对于以太网 HLEN = 6.             |
|              | ARP HTYPE, 硬件类型 (16 bit integer)                         | [*arp htype* «HTYPE»](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_headers#Matching_ARP_headers) | 无名称的16位长度的整数。<br/>[if_arp.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/if_arp.h)中列出已知类型. |
|              | ARP PLEN, 以字节为单位的互联网络地址长度 (8 bit integer)     | [*arp plen* «PLEN»](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_headers#Matching_ARP_headers) | 无名称的8位整数。<br/>对于IPv4而言， PLEN = 4.               |
|    arp_op    | ARP 操作类型 (16位长度整数, 数值使用预定义的常量，值是确定的):*request* = 1  *reply* = 2  *rrequest* = 3  *rreply* = 4  *inrequest* = 8  *inreply* = 9  *nak* = 10 | [*arp operation* «arp_op»](https://wiki.nftables.org/wiki-nftables/index.php/Matching_packet_headers#Matching_ARP_headers) |                                                              |



### IP types

|              |                                                              |                                                              |                                                              |
| :----------: | :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **数据类型** | **描述**                                                     | **表达式**                                                   | **备注**                                                     |
|  inet_proto  | 互联网协议 (8位长度整数,数值使用预定义的常量):*tcp* *udp* *udplite* *esp* *ah* *icmp* *icmpv6* *comp* *dccp* *sctp* | ip protocol<br/>ip6 nexthdr<br/>ah nexthdr<br/>comp nexthdr<br/>ct {original \|reply} protocol | [in.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/in.h) 中列出已知类型. |
| inet_service | 网络服务端口号(16位长度整数).                                | udp {sport \|dport}<br>tcp {sport \|dport}<br/>udplite {sport \|dport}<br/>sctp {sport \|dport}<br/>dccp {sport \|dport} |                                                              |
|  ipv4_addr   | IPv4 地址(32位长度整数).                                     | ip {saddr \|daddr}<br>arp {saddr \|daddr} ip<br>ct {original \|reply} ip {saddr \|daddr}<br>rt ip nexthop<br>ipsec {in \| out} ip {saddr \|daddr} |                                                              |
|  ipv6_addr   | IPv6 地址(128位长度整数).                                    | ip6 {saddr \|daddr}<br/>ct {original \| reply} ip6 {saddr \|daddr}<br/>rt ip6 nexthop<br/>ipsec {in \|out} ip6 {saddr \|daddr} |                                                              |

### Conntrack types

| **数据类型** | **描述**                          | **表达式** | **备注**                                                     |
| :----------: | --------------------------------- | :--------- | :----------------------------------------------------------- |
|    ct_dir    | 链接跟踪的方向(8位长度的整数).    |            | Symbolic constants:<br>original       0 <br/>reply          1 |
|   ct_event   | 链接跟踪事件位 (4位长度的位掩码). |            | Symbolic constants:<br>new            1 <br/>related        2<br/> destroy        4 <br/>reply          8 <br/>assured       16 <br/>protoinfo     32<br/> helper        64 <br/>mark         128 <br/>seqadj       256<br/> secmark      512<br/> label       1024 |
|   ct_label   | Conntrack label (128位的位掩码).  |            |                                                              |
|   ct_state   | Conntrack state(4字节位掩码).     |            | Symbolic constants:<br/>invalid        1 <br/>established    2 <br/>related        4<br/> new            8 <br/>untracked     64 |
|  ct_status   | Conntrack status (4字节位掩码).   |            | Symbolic constants:<br>expected       1<br> seen-reply     2 <br>assured        4 <br>confirmed      8<br/> snat          16 <br/>dnat          32 <br/>dying        512 |

