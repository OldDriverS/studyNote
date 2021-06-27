# 配置chain（链）

与iptables一样，使用nftables可以将规则（rule）附加到链(chain)上。与iptables不同，nftables没有预定义的INPUT、OUTPUT等链。相反，为了在特定的处理步骤中过滤数据包，您可以显式地创建一个带有您自定义名称的基链（**base chain**），并将它附加到适当的钩子（**hook**）上。这允许非常灵活的配置，而且不会因为你的规则集中包含了不需要的内置链减慢Netfilter。



## 添加基链（ base chain）

**base chains **是那些注册到Netfilter钩子（ [hooks](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks)）的链，也就是说，这些链检查流经你的Linux TCP/IP堆栈的数据包。

添加基链的语法是：

```bash
% nft add chain [<family>] <table-name> <chain-name> { type <type> hook <hook> priority <value> \; [policy <policy>] }
```



下面的例子展示了如何添加一个新的基链（链名称为input）输入到foo表(foo表在添加规则前创建完成):

```bash
% nft 'add chain ip foo input { type filter hook input priority 0 ; }'
```

重点：*nft* 重用特殊字符，如花括号`{}`和分号`;`。如果是从bash等shell运行这些命令，需要转义所有特殊字符。防止shell试图解析nft语法，最简单方法是在单引号包裹后续的参数。也可以执行该命令进入交互模式：

```bash
% nft -i
```

交互模式下运行非功能性测试。

*add chain* 添加 **input** 链,它绑定 *input hook* ,这个链将在input钩子检查点处，按其包含的规则处理流往本机的数据包。

*priority* （优先级）很重要，因为它决定了链的顺序。因此，如果你在 *input hook* 中有几个链,可以通过优先级属性的数值决定哪个链优先级更高。例如，**input**链的优先级-12、-1、0、10将按照该顺序进行查找。有可能两个基本链优先级相同,但没有保证的基本链的顺序,并将其固定在相同的钩子位置上。

如果想使用 *nftables* 来过滤桌面Linux计算机的流量（即不转发流量的计算机），也可以注册 *output* 链:

```bash
% nft 'add chain ip foo output { type filter hook output priority 0 ; }'
```

现在已经准备好过滤`传入的`(流往本地)和`流出的`(从本地往外)流量。

重要注意：

如果未在花括号中指定的链配置（未指定链的类型、对应的hook以及优先级），那么您创建的是一个常规链（**regular chain**）,它不会处理任何数据包(类似于 *iptables -N chain-name* )。常规链不会绑定到特定的hook上，它需要在基链中被跳转（jump）才能使用。

从nftables 0.5开始，你也可以像在iptables中一样为基链指定默认过滤策略（*policy*）:

```
% nft 'add chain ip foo output { type filter hook output priority 0 ; policy accept; }'
```

默认策略（*policy*）取值可能是 *accept* 和 *drop*，与iptables一致。

本链中所有的规则均匹配失败时，按默认策略处理数据包，若 *policy* 为 *drop* 将丢弃数据包，如为 *accept* 则该数据包继续流入到下个低优先级的基链继续遍历。当这个检查点（hook）中关联的所有基链规则均 *accept* 时，数据包进入下一个检查点（hook）,如果有一个基链拦截了数据包，那它将直接被抛弃，不进入后续的链中匹配规则。

**ingress** 钩子上添加链时，必须指定该链将被绑定网卡设备**eth0**:

```bash
% nft 'add chain netdev foo dev0filter { type filter hook ingress device eth0 priority 0 ; }'
```



### 基链的类型（Base chain types）

基链的类型可能是以下的：

- **filter** ，用于过滤数据包。arp、bridge、ip、ip6和inet协议簇族的表均支持。
- **route**，如果任何相关的IP报头（header）字段或包标记（packet mark）被修改，则用于重新路由数据包。如果熟悉iptables，则此链类型提供了与mangle表相同的语义，但仅用于 *output hook* (对于其他hook，应该使用*filter* 代替)。ip、ip6和inet协议簇的表均支持。
- **nat**，用于进行网络地址转换(NAT)。只有给定流的第一个包到达此链；随后的数据包将绕过它。因此，不要使用此链进行过滤。支持 *nat* 链类型的协议簇有ip、ip6和inet。

### 基链的钩子（Base chain hooks）

当你配置你的基链时，需要指定hook属性将基链绑定到特定的hook，可能使用的值是:

- **ingress** (仅在Linux内核4.2之后的netdev系列中，在Linux内核5.10之后的inet系列中):可以在数据包从网卡驱动上传递后立即检查数据包，甚至在**prerouting**之前。它可以作为*tc*的代替品。
- **prerouting** 在经过路由表查询前，检查所有传入的数据包。数据包可以发送到本地或转发到远程系统。
- **input** 检查发送到本地并已完成路由处理的传入数据包。
- **forward** 检查已过路由、但并未定位到本地系统的传入数据包。
- **output** 检查来自本机进程的数据包。
- **postrouting** 检查已经完成路由后的所有数据包，就在数据包即将离开本地系统的检查点。



### 基链的优先级（Base chain priority）

每个基链都被分配了一个优先级属性（*priority*），*priority* 定义了它在同一 *hook* 上的其他基链、流表和**Netfilter**内部操作之间的顺序。例如，一个优先级为-300的prerouting钩子上的链将被放置在连接跟踪操作（connection tracking，简写*ct*）之前。

注意：

如果一个数据包被接受，并且有另一个链，具有相同的 *hook* 类型和稍后的优先级，那么该数据包随后将遍历另一个链。

因此，一个被*accept* 的动作（无论是通过规则还是默认链策略），不一定是最终的。

在高优先级的基链中被*accept*的数据包，可能因为继续遍历低优先级的基链中因被*dop* （可能因为匹配规则被drop或者因为默认策略被drop）最终导致这个数据包被丢弃。

反而被drop立即生效的，它不会再进入到低优先级的链中去。因此在编写规则的时候，要注意同一个hook下但是不同优先级链遍历的问题。

下面的规则集演示了这种行为，令人意想不到：

```ini
table inet filter {
        # 由于services链优先级较高，先被遍历匹配。
        chain services {
                type filter hook input priority 0; policy accept;

                # 如果匹配则丢弃数据包
                tcp dport http drop

                # 如果匹配，则accept后，还要进入下一个优先级较低的input中再遍历。
                tcp dport ssh accept

                # 最终没有被匹配的数据包，执行默认的策略，也就是accept，也将进入下一个低优先级的链中继续遍历规则。
        }

        # input 链的优先级较低
        chain input {
                type filter hook input priority 1; policy drop;
                # 但是最终所有的数据包都会被抛弃
                # 即使在services链中已经被accept,它还是要遍历低优先级的input链
                # 最终input链的默认规则是drop，拦截数据包。
        }
}
```

若上面的**input**链的优先级被更改为-1（即使得**input**链相较**services**链更优先），区别是：没有数据包有机会进入**services**链。但是无论哪种方式，该规则集都会导致所有进入的数据包被丢弃。

总之，包将遍历给定钩子范围内的所有链，直到它们被丢弃或不再存在基链。只有在没有后面的链承载与数据包最初进入的链相同类型钩子的情况下，接受裁决（verdict）才保证是最终的。



### 基链的默认策略（Base chain policy）

这是将应用于到达链的末端的数据包的默认判定(即在该链已经完成了所有规则匹配后均无符合条件的)。

目前有两种策略: 

- **accept** (默认)  意味着数据包将继续遍历网络堆栈(默认)。
- **drop** 表示报文到达基链的末端时被丢弃。

注意：如果没有显式定义*policy*，将默认使用accept*。



## 添加规则链（Adding regular chains）

你也可以创建规则链，类似于iptables用户定义链：

```
% nft add chain ip <table_name> <chain_name>
```

 <chain_name> 链名是任意大小写的任意字符串。

注意，在添加规则链(**regular chain**)时，不包括 *hook钩子* 关键字。因为它没有绑定到 *hook*，规则链本身是不检查任何流量的。

但是一个或者多个基链可以通过*jump*跳转的动作去包含这个规则链，使得该链的规则被遍历。

除此之外，常规链以与调用基链完全相同的方式处理包。

通过使用jump和/或goto操作，将你的规则集将基础链和规则链排列成树结构是非常有用的。(虽然我们有点超前了，但nftables vmap提供了一种更强大的方法来构建高效的分支规则集。)





## 链的删除（Deleting chains）

可以使用以下方式删除一个链：（以删除ip协议簇的foo表中的input链为例）

```bash
% nft delete chain ip foo input
```

唯一的条件是要删除的链必须是空的，否则内核会反馈该链仍在使用中。

```bash
% nft delete chain ip foo input
<cmdline>:1:1-28: Error: Could not delete chain: Device or resource busy
delete chain ip foo input
^^^^^^^^^^^^^^^^^^^^^^^^^
```

在删除链之前，您必须刷新该链中的规则集（[flush the ruleset](https://wiki.nftables.org/wiki-nftables/index.php/Simple_rule_management)）。



## 刷新链（Flushing chains）

刷新foo表的链输入(删除所有规则)：

```
nft flush chain foo input
```



# 示例配置:为单机过滤流量

你可以创建一个包含两个基链的表过滤进出你的计算机的流量的规则，假设是IPv4连接:

```
% nft add table ip filter
% nft 'add chain ip filter input { type filter hook input priority 0 ; }'
% nft 'add chain ip filter output { type filter hook output priority 0 ; }'
```



 现在，你可以开始给这两个基链附加规则了。注意，在本例中不需要正向链，因为本例假设您正在配置nftables来过滤不具有路由器行为的单机的流量。

