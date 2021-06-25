# Netfilter hooks

**nftables**使用的**Netfilter**基础架构与遗留的iptables基本相同。钩子基础结构、[连接跟踪系统](http://people.netfilter.org/pablo/docs/login.pdf)、NAT引擎（NAT engine）、日志基础结构（logging infrastructure）和用户空间队列（userspace queueing）保持不变。只有数据包分类框架（packet classification framework）是新的。



## Netfilter hooks into Linux networking packet flows

以下示意图显示了通过Linux网络的数据包流:

[![nf-hooks](nf-hooks.png)

在**input**路径中流向本地机器的流量会看到**prerouting**钩子和**input**钩子。然后，本地进程生成的流量遵循**output**路径和**postrouting**路径。

如果你把你的Linux机器配置成一个路由器，不要忘记通过以下方式启用转发:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

然后从**forward**钩子中可以看到未发送到本地系统的数据包。这些被转发的数据包遵循以下路径:**prerouting**、**forward**和**postrouting**。

## Ingress hook（入口钩子）

**ingress **钩子是在Linux内核4.2中添加的。与其他钩子不同的是，**ingress**钩子是绑定到特定的网络接口（network interface）上的。

可以使用nftables和**ingress**钩子来执行甚至在**prerouting**之前生效的过滤策略。请注意，在这个非常早期的阶段，碎片数据报还没有被重新组装。所以，例如，匹配IP源地址（ip saddr）和目的地址（daddr）对所有的ip数据包都有效，但是匹配四层协议头（L4 headers）如`udp dport`只对未分片的数据包（unfragmented packets）有效，或第一个片段（first fragment）。

**ingress **钩子提供了一种替代tc输入过滤的方法。但是您仍然需要tc来进行流量整形/队列管理（shaping/queue）。

# 钩子相关的协议簇(family)和链类型(chain type)


下表列出了可用的协议簇(family)和链类型(chain type)。最小的nftable和Linux内核版本显示了最近添加的钩子。



|     Hooks     |                                                              |            |         |       |        |             |                                                |
| :-----------: | ------------------------------------------------------------ | ---------- | ------- | ----- | ------ | ----------- | ---------------------------------------------- |
|               | **ingress**                                           | **prerouting** | **forward** | **input** | **output** | **postrouting** | **egress**                               |
|  **inet family**  ||||||||
|    filter     | [0.9.7](https://marc.info/?l=netfilter&m=160379555303808&w=2) / [5.10](https://kernelnewbies.org/Linux_5.10) | Yes        | Yes     | Yes   | Yes    | Yes         | No                                             |
|      nat      | No                                                           | Yes        | No      | Yes   | Yes    | Yes         | No                                             |
|     route     | No                                                           | No         | No      | No    | Yes    | No          | No                                             |
| **ip6 family** |                                                              |            |         |       |        |             |                                                |
|    filter     | No                                                           | Yes        | Yes     | Yes   | Yes    | Yes         | No                                             |
|      nat      | No                                                           | Yes        | No      | Yes   | Yes    | Yes         | No                                             |
|     route     | No                                                           | No         | No      | No    | Yes    | No          | No                                             |
|   **ip family**   |                                                              |            |         |       |        |             |                                                |
|    filter     | No                                                           | Yes        | Yes     | Yes   | Yes    | Yes         | No                                             |
|      nat      | No                                                           | Yes        | No      | Yes   | Yes    | Yes         | No                                             |
|     route     | No                                                           | No         | No      | No    | Yes    | No          | No                                             |
| **arp family** |                                                              |            |         |       |        |             |                                                |
|    filter     | No                                                           | No         | No      | Yes   | Yes    | No          | No                                             |
|      nat      | No                                                           | No         | No      | No    | No     | No          | No                                             |
|     route     | No                                                           | No         | No      | No    | No     | No          | No                                             |
| **bridge family** |                                                              |            |         |       |        |             |                                                |
|    filter     | No                                                           | Yes        | Yes     | Yes   | Yes    | Yes         | No                                             |
|      nat      | No                                                           | No         | No      | No    | No     | No          | No                                             |
|     route     | No                                                           | No         | No      | No    | No     | No          | No                                             |
| **netdev family** |                                                              |            |         |       |        |             |                                                |
|    filter     | [0.6](https://marc.info/?l=netfilter&m=146488681521497&w=2) / [4.2](https://kernelnewbies.org/Linux_4.2) | No         | No      | No    | No     | No          | - / [5.7](https://kernelnewbies.org/Linux_5.7) |
|      nat      | No                                                           | No         | No      | No    | No     | No          | No                                             |
|     route     | No                                                           | No         | No      | No    | No     | No          | No                                             |



##  hook的优先级

在一个给定的钩子中，按照增加数值优先级的顺序执行操作。每个基链（ [base chain](https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains#Base_chain_hooks)）和流表（[flowtable](https://wiki.nftables.org/wiki-nftables/index.php/Flowtables)）都被分配了一个优先级，该优先级定义了它在同一钩子上的其他**base chain**和**flowtable**在Netfilter内部操作之间的顺序。例如，一个优先级为-300的**prerouting**钩子上的链将被放置在连接跟踪（**connection tracking**）操作之前。

> 数值越低优先级越高

Netfilter优先级值如下表所示，请参考nft手册页。

| nftables [Families](https://wiki.nftables.org/wiki-nftables/index.php/Nftables_families) | Typical hooks      | *nft* Keyword |   Value | Netfilter Internal Priority | Description                                                  |
| :----------------------------------------------------------- | :----------------- | :------------ | ------: | :-------------------------- | :----------------------------------------------------------- |
|                                                              | prerouting         |               |    -450 | NF_IP_PRI_RAW_BEFORE_DEFRAG |                                                              |
| inet, ip, ip6                                                | prerouting         |               |    -400 | NF_IP_PRI_CONNTRACK_DEFRAG  | 包碎片整理/数据报重组                                        |
| inet, ip, ip6                                                | all                | **raw**       |    -300 | NF_IP_PRI_RAW               | 传统的将原始表的优先级放在连接跟踪操作之前                   |
|                                                              |                    |               |    -225 | NF_IP_PRI_SELINUX_FIRST     | SELinux 操作                                                 |
| inet, ip, ip6                                                | prerouting, output |               |    -200 | NF_IP_PRI_CONNTRACK         | [Connection tracking（连接跟踪）](https://wiki.nftables.org/wiki-nftables/index.php/Connection_Tracking_System) 处理在**prerouting**和**output**钩子的早期运行，以将数据包与被跟踪的连接关联起来。 |
| inet, ip, ip6                                                | all                | **mangle**    |    -150 | NF_IP_PRI_MANGLE            | Mangle 操作                                                  |
| inet, ip, ip6                                                | prerouting         | **dstnat**    |    -100 | NF_IP_PRI_NAT_DST           | Destination NAT(目的地址转换)                                |
| inet, ip, ip6, arp, netdev                                   | all                | **filter**    |       0 | NF_IP_PRI_FILTER            | 过滤操作，过滤表                                             |
| inet, ip, ip6                                                | all                | **security**  |      50 | NF_IP_PRI_SECURITY          | 安全表设置,例如可以设置secmark                               |
| inet, ip, ip6                                                | postrouting        | **srcnat**    |     100 | NF_IP_PRI_NAT_SRC           | SNAT(源地址转换)                                             |
|                                                              | postrouting        |               |     225 | NF_IP_PRI_SELINUX_LAST      | SELinux 包出口                                               |
| inet, ip, ip6                                                | postrouting        |               |     300 | NF_IP_PRI_CONNTRACK_HELPER  | 连接跟踪（ct helper），用于识别预期的和相关的数据包。        |
| inet, ip, ip6                                                | input, postrouting |               | INT_MAX | NF_IP_PRI_CONNTRACK_CONFIRM | 连接跟踪在input & poststrouting钩子的最后一步添加新的跟踪连接。 |
|                                                              |                    |               |         |                             |                                                              |
| bridge                                                       | prerouting         | **dstnat**    |    -300 | NF_BR_PRI_NAT_DST_BRIDGED   |                                                              |
| bridge                                                       | all                | **filter**    |    -200 | NF_BR_PRI_FILTER_BRIDGED    |                                                              |
| bridge                                                       |                    |               |       0 | NF_BR_PRI_BRNF              |                                                              |
| bridge                                                       | output             | **out**       |     100 | NF_BR_PRI_NAT_DST_OTHER     |                                                              |
| bridge                                                       |                    |               |     200 | NF_BR_PRI_FILTER_OTHER      |                                                              |
| bridge                                                       | postrouting        | **srcnat**    |     300 | NF_BR_PRI_NAT_SRC           |                                                              |



从nftables 0.9.6开始，您可以使用关键字而不是数字来设置优先级。(注意，在**bridge**协议簇相对于其他协议簇，相同的关键字映射到不同的数字优先级)您也可以指定优先级作为关键字的整数偏移量，例如，mangle - 5等价于数字优先级的-155。

关于用关键字来代替数值的笔记：
**nft  Keyword**列中的关键字能代替对应Value的数值，但是这个数值针对不同表不同类型的基链，具体的数值也会发生变化

```
sudo nft add chain ip nat my-prerouting  { type nat hook prerouting  priority dstnat \; } 
# 等价
sudo nft add chain ip nat my-prerouting  { type nat hook prerouting  priority -100 \; } 
```

