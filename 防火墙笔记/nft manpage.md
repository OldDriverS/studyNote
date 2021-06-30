# nft 帮助手册

## 名称

*nft* - 用于包过滤和分类的nftables框架的管理工具

## 大纲

**nft**
[ **-nNscae** ] [ -I *directory* ] [ -f *filename* | -i | *cmd* ...]

**nft -h**

**nft -v**





## 描述

获取选项的完整摘要，请运行 **nft --help**。

**-h, --help**

​    显示帮助信息和所有选项。

**-v, --version**

​    显示版本信息。

**-n, --numeric**

​    显示数值的形式显示数据。当使用1次(默认行为)时，跳过对符号名的地址查找。使用2次也可以以数字形式显示网络服务(端口号)。使用三次则可以以数值形式显示协议和UID/GID。

**-N, --reversedns**

​    将IP地址转换为名称。通常需要网络流量进行DNS查找。

**-s, --stateless**

​    省略规则和有状态对象（ **stateful objects** ）的有状态信息。

**-c, --check**

​    仅检查命令的有效性，而不实际应用更改。

**-a, --handle**

​    在输出中显示对象句柄（*handle*）。

**-e, --echo**

在规则集中插入项时使用 **add**, **insert** 或 **replace** 命令, 就像 **nft monitor** 一样回显结果通知。

**-I, --includepath** *directory*

​	将目录 *directory* 添加到文件导入的目录搜索列表中。这个选项可以多次指定。

**-f, --file** *filename*

​    从 *filename* 指定的文件名读取输入。如果 *filename* 是 **-**，从输入流（*stdin*）读取。

​    **nft** 脚本文件中的第一行必须添加 **#!/usr/sbin/nft -f** 。以指定nft作为脚本的解释器。

**-i, --interactive**

​    从交互式命令行中读取输入。您可以使用 **quit** 退出，或者使用 **EOF** 标记，通常是 *CTRL-D* 。



## 输入文件格式（nft 脚本文件格式）

### 词法规定

输入被逐行解析（line-wise），当换行符之前的最后一个字符是非引号反斜杠（\\）时，下一行将被视为延续。同一行中的多个命令可以用分号（;）分隔。

- 注释以#开头。忽略同一行上的所有字符。

- 标识符以字母字符(a-z, a-z)开头，后面跟着0个或多个字母数字字符(a-z, a-z, 0-9)，以及字符斜杠(/)、反斜杠(\\)、下划线(_)和点(.)。使用不同字符或与关键字冲突的标识符需要用双引号括起来(")。

    

### 导入文件

**include** *filename*

可以使用 **include** 语句包含其他文件。要从特定的目录搜索文件可以使用 **-I/--includepath** 选项指定目录。你可以通过在你的路径前加上 ./ 来强制包含位于当前工作目录下的文件(即相对路径)，或 / 开头的路径的文件位置表示为绝对路径。

如果没有指定 **-I/--includepath**，那么 **nft** 依赖于编译时指定的默认目录。您可以通过 **-h/--help** 选项检索这个默认目录。

**include** 语句支持通常的shell通配符符号 (*, ?, [])。如果 **include** 语句中使用了通配符，则包含语句中没有匹配项不会出错。这允许为 include "/etc/firewall/rules/\*" 句提供潜在的空导入目录。通配符匹配按字母顺序加载。以点(.)开头的文件不会由include语句匹配。



### 符号变量

**define** variable *expr*

**$variable**

符号变量可以使用define语句来定义。变量引用是表达式，可以用来初始化其他变量。定义的作用域是当前块和包含在其中的所有块。

**使用符号变量**

```nft
define int_if1 = eth0
define int_if2 = eth1
define int_ifs = { $int_if1, $int_if2 }
filter input iif $int_ifs accept
```



## 地址协议簇（ADDRESS FAMILIES）

地址簇决定了被处理的数据包的类型。对于每个地址簇，内核在包处理路径的特定阶段包含所谓的钩子（*hook*），如果存在这些钩子的规则，这些钩子将调用 nftables。



| 名称 | 说明 |
| :-: | ---- |
| **ip** | IPv4 地址协议簇 |
| **ip6** | IPv6 地址协议簇 |
| **inet** | Internet (IPv4/IPv6)地址族。 |
| **arp** | ARP地址协议簇，处理IPv4 ARP报文。 |
| **bridge** | 网桥地址族，处理通过网桥设备的数据包。 |
| **netdev** | Netdev地址族，处理来自入接口（ *ingress* ）的数据包。 |

所有 nftables 对象都存在于特定于地址簇的名称空间中，因此所有标识符都包括一个地址族。如果不指定地址族，则默认使用ip族。



### IPV4/IPV6/INET协议簇

IPv4/IPv6/Inet 地址簇处理IPv4、IPv6或这两种类型的数据包。它们在网络堆栈的不同包处理阶段包含五个钩子（*hook*）。

**IPv4/IPv6/Inet 地址协议簇相关的 hooks**

| Hook        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| prerouting  | 所有进入系统的数据包都由 prerouting 钩子处理。它在路由处理之前被调用，用于早期过滤或更改影响路由的包属性。 |
| input       | 发送到本地系统的数据包由 input 钩子处理。                    |
| forward     | forward 钩子处理转发到不同主机的报文。                       |
| output      | 本地进程发送的数据包由 output 钩子处理。                     |
| postrouting | 所有离开系统的数据包都由 poststrouting 钩子处理。            |



### ARP地址协议簇

ARP 地址协议簇用于处理系统接收和发送的ARP报文。它通常用于修改ARP包以实现集群化。

**ARP地址协议簇相关的 hooks**

| Hook   | Description                               |
| ------ | ----------------------------------------- |
| input  | 发送到本地系统的数据包由 input 钩子处理。 |
| output | 本地进程发送的数据包由 output 钩子处理。  |



### BRIDGE 地址协议簇

BRIDGE 地址协议簇处理通过网桥设备的以太网（Ethernet）数据包。

支持的 *hook* 列表与上述IPV4/IPV6/INET协议簇相同。

### NETDEV 地址协议簇

Netdev 地址协议簇处理来自入接口的数据包。

**NETDEV 地址协议簇相关的 hooks**

| Hook    | Description                                                  |
| ------- | ------------------------------------------------------------ |
| ingress | 所有进入系统的数据包都由这个 hook 处理。它在第3层协议（layer 3 protocol）处理程序之前被调用，可以用于早期过滤和监管 |



## 规则集（RULESET）

{ list | flush } **ruleset** [ *family* ] 

export [ **ruleset** ] *format*



| 命令       | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| **list**   | 以阅读友好的格式打印规则集。                                 |
| **flush**  | 清空整个规则集（ruleset）。请注意，与iptables不同的是，这将删除所有表（table）及其包含的任何内容，导致一个空的规则集生效——空的规则集将导致不再进行包过滤，因此内核接受它接收到的任何有效包。 |
| **export** | 以机器可读的格式打印规则集。*format*  参数可以是 **xml** 或 **json** 。 |

可以只将 **list** 和 **flush** 操作限制作用于特定的协议簇（family）。有关有效的 *family* 名称列表，请参阅上面的**地址协议簇（ADDRESS FAMILIES）**。



## 表（TABLES）

{ add | create } **table** [ *family* ] *table* [ { flags *flags* } ]

{ delete | list | flush } **table** [ *family* ] *table*

delete **table** [ *family* ] handle *handle*

表是链（chains）、集合（sets）和有状态对象（stateful objects）的容器。表通过通过地址簇与名字来区分。表的协议簇（ *family* ）必须是 **ip**, **ip6**,  **inet**, **arp**,  **bridge**, **netdev**中的其中一个。**inet** 地址簇是一个虚拟的地址族，用于创建IPv4/IPv6混合表。元表达式（meta expression）*nfproto* 关键字可用于测试数据包正在哪个簇(IPv4或IPv6)的上下文（context）中处理。当未指定地址簇时，默认为 **ip**。**add** 和 **create** 操作之间的唯一区别是，如果指定的表 *table* 已经存在， **add** 不会返回错误，而 **create**将返回错误。

**Table 相关标志(flag)**

| Flag    | 说明                                            |
| ------- | ----------------------------------------------- |
| dormant | 表不再计算(基链被卸载) （译注：暂时禁用这个表） |



**Add, change, delete a table**

```bash
# 进入nft交互模式
nft --interactive
# 创建一个新的表mytable.
create table inet mytable
# 添加一个新的基链: 链类型为filter，指定input钩子，优先级数值为0。用于匹配输入的数据包
add chain inet mytable myin { type filter hook input priority 0; }
# 向链中添加一个计数器（counter）
add rule inet mytable myin counter
# 暂时禁用这个表-- 规则将不会被计算。
add table inet mytable { flags dormant; }
# 再次激活这个表。
add table inet mytable
```



| 表操作命令 | 描述                                                         |
| :--------: | ------------------------------------------------------------ |
|  **add**   | 为具有给定名称（ *table* ）及协议簇（ *family* ）添加一个新表。 |
| **delete** | 删除指定的表。                                               |
|  **list**  | 列出指定表的所有链和规则。                                   |
| **flush**  | 清理指定表的所有链和规则。                                   |

## 链（CHAINS）

{ add | create } **chain** [ *family* ] *table* *chain* [ { type *type* hook *hook* [ device *device* ] priority *priority*;  [ policy *policy*; ] } ]

{ delete | list | flush } **chain** [ *family* ] *table* *chain*

delete **chain** [ *family* ]  *table* handle *handle*

rename **chain** [ *family* ] *table* *chain* *newname*

链（**chain**）是规则（**rule**）的容器。有两种类型的链：基链（**base chains**）与规则链（**regular chains**）。基链是来自网络堆栈的数据包的入口点，规则链可以用作跳转目标，用于更好地组织规则。

| 链操作命令 | 说明                                                         |
| :--------: | ------------------------------------------------------------ |
|  **add**   | 在指定的表中添加一个新链。当指定了钩子（*hook*）和优先级值（*priority*）时，该链被创建为基链，并与网络堆栈（ *networking stack* ）连接起来。 |
| **create** | 类似于 *add* 命令，但如果链已经存在，则返回错误。            |
| **delete** | 删除指定的链。该链不能包含任何规则或作为被跳转（jump）目标。 |
| **rename** | 重命名指定的链。                                             |
|  **list**  | 列出指定链的所有规则。                                       |
| **flush**  | 清空指定链的所有规则。                                       |

对于基链，**type**， **hook** 和 **priority**  参数是必需的。

基链支持的类型（type）有

| 类型（*Type*） | 簇（Families） |           钩子<br>（Hooks）            | 描述                                                         |
| :------------: | :------------: | :------------------------------------: | ------------------------------------------------------------ |
|     filter     |      all       |                  all                   | 在标准链型使用是存疑的。                                     |
|      nat       |    ip, ip6     | prerouting, input, output, postrouting | 这种类型的链基于连接跟踪表项（conntrack entries）执行网络地址转换（ Network Address Translation）。只有连接的第一个包真正遍历这个链—它的规则通常定义创建的连接跟踪条目的细节(例如NAT语句)。 |
|     route      |    ip, ip6     |                 output                 | 如果一个包已经遍历了这种类型的链并且即将被接受，那么如果IP报头的相关部分发生了变化，将执行一个新的路由查找。例如，这将允许在nftables中实现策略路由选择器。 |

除了上述的特殊情况（例如 *nat* 类型不支持 **forward** 钩子，**route** 类型只支持 **output** 钩子），还有两个特殊情况值得注意：

- netdev 簇只支持单一的组合，即 *type filter* 和  *hook ingress*。这个family中的基链也要求设备参数存在，因为它们只存在于每个网络设备入口（incoming interface）上。

- arp 簇只支持 *input* 和 *output* 钩子，都是 *filter* 类型链中的 hook。

**priority** 参数接受一个带符号的整数值，在相同的 hook 下的链，它们将决定按优先级的值（数值越小优先级越高）的链的遍历顺序，优先级数值上可以是负数的，优先级高的链优先被访问。

基础链允许设置它的默认策略（ **policy** ），例如，在包含的规则中未显式接受或拒绝的数据包会发生什么？支持的策略（**policy** ）值是 *accept* (默认值)或 *drop* 。（译注：当当前的链中所有规则都被遍历完毕后，仍未被匹配，则按policy指定的策略处理。）



### 规则（RULES）

[ add | insert ] **rule** [ *family* ] *table* *chain* [ { handle | position } *handle* | index *index* ] *statement*...

replace **rule** [ *family* ] *table* *chain* handle *handle* *statement*...

delete **rule** [ *family* ] *table* *chain* handle *handle*



规则被添加到给定表中的链（chain）中。如果不指定 *family*，则默认为 **ip** 地址簇。规则（rules）是根据一套语法规则由两种成分构成的：表达式（expressions ）和语句（statements）。

**add** 和 **insert** 命令支持一个可选的位置指示符，它要么是现有规则的句柄（ *handle* ），要么是一个范围从0开始的索引（ *index*）。在内部，规则位置总是由 *handle* 确定，而且 *handle* 转换成 *index* 的转换发生在用户空间。如果在转换完成后发生并发规则集（ruleset ）更改，这有两个潜在的含义：

- 如果在引用的规则之前插入或删除了规则，则有效的规则索引值可能会发生变化。
- 如果删除被引用的规则，此条命令将被内核会拒绝，就像抛出了一个无效的 *handle* 一样。

| 规则操作命令 | 说明                                                         |
| :----------: | ------------------------------------------------------------ |
|   **add**    | 添加一个由语句序列（**statements**）描述的新规则。如未指定 **handle**，规则将被追加至指定链的规则列表末端，若指定**handle**，在这种情况下，新规则添加到由 **handle** 指定的规则位置之后。替代名称位置（alternative name position）已弃用，不应再使用。 |
|  **insert**  | 类似于 **add** 命令，但该规则被添加到链的开头或 **handle** 指定的规则之前。 |
| **replace**  | 类似于 **add **命令，但该规则将替换指定的规则。              |
|  **delete**  | 删除指定的规则。                                             |

向 **ip** 表 **input** 链添加规则

```bash
# 表是通过 地址簇+表名称确定的，nftables没有内置的表，需要先创建。
# 如果规则新增命令中没有包含相关的family，则假定为ip簇，即filter表等价于ip filter表。
nft add rule filter output ip daddr 192.168.0.0/24 accept 
# 下面是上面相同的命令，只是指定了表为ip input，更详细一些。
nft add rule ip filter output ip daddr 192.168.0.0/24 accept
```

从 **inet** 表中删除规则

```bash
# nft -a list ruleset
table inet filter {
        chain input {
                type filter hook input priority 0; policy accept;
                ct state established,related accept # handle 4
                ip saddr 10.1.1.1 tcp dport ssh accept # handle 5
		...
# 以下命令为删除 handle 为 5 的规则。
# nft delete rule inet filter input handle 5
```





## 集合（SETS）

Nftables提供了两种集合概念。匿名集（Anonymous sets）是没有特定名称的集。在创建使用集合的规则时，集合成员用花括号（ *{ }* ）括起来，用逗号（ *,* ）分隔元素。一旦该规则被删除，匿名集也将被删除。它们不能被更新，也就是说，一旦匿名集合被声明，它就不能再被更改，除非 删除/更改 使用匿名集合的规则（rule）。

使用匿名集来接受特定的子网和端口

```bash
nft 'add rule filter input ip saddr { 10.0.0.0/8, 192.168.0.0/16 } tcp dport { 22, 443 } accept'
```

命名集（Name sets）是在规则中引用它们之前需要先定义的集合。与匿名集不同，元素可以在任何时候添加或删除命名集。集合使用前缀为集合名的 **@** 从规则中引用。

使用命名集来放行地址和端口

```bash
 nft add rule filter input ip saddr @allowed_hosts tcp dport @allowed_ports accept
```

首先需要创建  *allowed_hosts* 和 *allowed_ports* 集合。下一节将更详细地描述 **nft** 集合（**Set**）相关的语法。

add **set**  [ *family* ] *table* *set* { type *type*; [ flags *flags*; ] [timeout *timeout*; ] [ gc-interval *gc-interval*; ] [ elements = { *element*[ ,... ] }; ] [ size *size*; ] [ policy *policy*; ] [ auto-merge *auto-merge*; ] }

{delete | list | flush } **set** [ *family* ] *table* *set*

delete **set** [ *family* ] *table* handle *handle*

{ add | delete } **element** [ *family* ] *table* *set* { *element*[ ,... ] }

| 操作集合的命令     | 说明                                       |
| ------------------ | ------------------------------------------ |
| **add**            | 在指定的表（table）中添加一个新集合。      |
| **delete**         | 删除指定的集。                             |
| **list**           | 显示指定集合中的元素。                     |
| **flush**          | 从指定的集合中删除所有元素。               |
| **add element**    | 将以逗号分隔的元素列表添加到指定的集合。   |
| **delete element** | 将以逗号分隔的元素列表从指定的集合中删除。 |



**Set 规范**

| 关键字      | 描述                                                         | 值类型                                                       |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| type        | 元素的数据类型                                               | 字符串：ipv4_addr, ipv6_addr, ether_addr, inet_proto, inet_service, mark |
| flags       | 集合的标志（flag）                                           | 字符串: constant, interval, timeout                          |
| timeout     | 元素停留在集合中的时间，如果从报文路径(ruleset)中添加，则为必选。 | 字符串, 10进制后加上单位. 单位的可选值是以下字符: d, h, m, s |
| gc-interval | 垃圾收集间隔，仅在 *timeout* 或 *timeout* 标志被激活时可用   | 字符串, 10进制后加上单位. 单位的可选值是以下字符:  d, h, m, s |
| elements    | 集合中包含的元素                                             | set data type                                                |
| size        | 集合中的最大元素数量，如果从数据包路径(规则集)中加入，则为必选。 | 无符号整数 (64位长度)                                        |
| policy      | 集合的默认策略                                               | 字符串： performance [default], memory                       |
| auto-merge  | 自动合并相邻/重叠集元素(仅适用于间隔集)                      |                                                              |



## 映射表（MAPS）

add **map** [*family*] *table* *map* { type *type* [flags *flags* ;] [elements = { *element*[,...] } ;] [size *size* ;] [policy *policy* ;] }

{delete | list | flush} **map** [*family*] *table* *map*

{add | delete} **element** [*family*] *table* *map* { elements = { *element*[,...] } ; }

映射表（Maps）基于作为输入的特定键存储数据，它们由用户定义的名称唯一标识并附加到表。

**Map 规范** 

| 关键字   | 描述                   | 值类型                                                       |
| -------- | ---------------------- | ------------------------------------------------------------ |
| type     | 映射元素的数据类型     | string ':' string: ipv4_addr, ipv6_addr, ether_addr, inet_proto, inet_service, mark, counter, quota. Counter and quota can't be used as keys |
| flags    | 映射表的标识配置       | string: constant, interval                                   |
| elements | 映射表中包含的元素     | 映射表元素的数据类型。                                       |
| size     | 映射表中的最大元素个数 | 无符号整数 (64 bit)                                          |
| policy   | 映射表的默认策略       | string: performance [default], memory                        |



## 流表（FLOWTABLES）

{ add | create } **flowtable** [ *family* ] *table* *flowtable* { hook *hook* priority *priority* ; devices = { *device*[,...] } ; }

{ delete | list } **flowtable** [ *family* ] *table* *flowtable*

流表（flowtable）允许您在软件中加速数据包转发。流表的条目（entry）通过一个元组（tuple）表示，该元组由输入接口（input interface）、源地址及目的地址（source and destination address）、源端口和目的端口（source and destination port）组成;属于3/4层协议。每个条目还缓存了目的网络接口和网关地址。用于更新目的链路层地址。用于转发数据包。TTL数值及hoplimit字段会被衰减。因此，流表提供了另一种路径，允许数据包绕过传统的转发路径。

流表位于 *ingress* 钩子中，它的位置要早于 *prerouting* 钩子。您可以从转发链中通过流卸载表达式选择要卸载的流（flow）。流表由其地址簇及名称作为区分标识。地址簇必须是**ip**,  **ip6**,  **inet** 的其中一个。**inet** 地址簇是一个虚拟的地址族，用于创建IPv4/IPv6混合表。当未指定地址簇时，默认为 **ip**。



| 有关流表的操作命令 | 描述                                       |
| :----------------: | ------------------------------------------ |
|      **add**       | 为具指定的表名及地址簇族添加一个新的流表。 |
|     **delete**     | 删除指定的流表。                           |
|      **list**      | 列出所有流表。                             |









## 有状态对象（STATEFUL OBJECTS）



{ add | delete | list | reset } **type** [ *family* ] *table* *object*

delete **type** [ *family* ] *table* handle *handle*



有状态对象（Stateful objects）被附加到表上，并由唯一的名称标识。

它们将规则中的有状态信息分组，若要在规则中引用它们，使用关键字格式 “type name”，例如 “counter name”。

| 有状态对象相关操作命令 | 说明                                 |
| ---------------------- | ------------------------------------ |
| **add**                | 在指定的表中添加一个新的有状态对象。 |
| **delete**             | 删除指定对象                         |
| **list**               | 显示对象持有的状态信息。             |
| **reset**              | 列出及重置有状态对象                 |



## 连接跟踪（ct）

**ct**
helper *helper* { type *type* protocol *protocol* ; [ l3proto *family*; ] }

**ct helper** 用于定义连接跟踪辅助工具（helper），然后可以与 **ct helper set** 语句结合使用。类型和协议是强制性的，默认情况下，*protocol 派生自表的簇。例如，在 **inet** 簇的表中，内核将尝试加载 **IPv4** 和 **IPv6** helper后端，如果它们被内核支持。

**conntrack helper 规范**

| 关键字（Keyword） | 描述               | 类型（Type）                 |
| ----------------- | ------------------ | ---------------------------- |
| type              | helper 类型的名称  | 被引用的字符串(例如： "ftp") |
| protocol          | helper的4层协议    | 字符串 (例如： tcp)          |
| l3proto           | helper的第三层协议 | 地址簇(例如 **ip** )         |

定义和分配 ftp helper

与iptables不同的是，helper分配需要在连接跟踪查找完成之后执行，例如，用于hook优先级值为0的链。

```
table inet myhelpers {
  ct helper ftp-standard {
     type "ftp" protocol tcp
  }
  chain prerouting {
      type filter hook prerouting priority 0;
      tcp dport 21 ct helper set "ftp-standard"
  }
}
```



## 计数器（COUNTER）

**counter**
[ packets bytes ]

**Counter 规范**

| Keyword | Description    | Type                |
| ------- | -------------- | ------------------- |
| packets | 数据包初始计数 | 无符号整数 (64 bit) |
| bytes   | 初始字节数计数 | 无符号整数 (64 bit) |



##  配额（QUOTA）

**quota**
[over | until] [used]



**Quota 规范**

| 关键字 | 描述                     | 类型                                                         |
| ------ | ------------------------ | ------------------------------------------------------------ |
| quota  | 配额限制，用作配额限制的 | 两个参数，无符号整数(64位)和字符串：**bytes** ,  **kbytes** , **mbytes**。"over" 及 "until" 需放在这两个参数组合之前。 |
| used   | 已用配额的初值           | 两个参数，无符号整数(64位)和字符串： **bytes** ,  **kbytes** , **mbytes** |



## 表达式（EXPRESSIONS）

表达式可以用值，如网络地址、端口号等常量表示。或在规则集审计期间从数据包收集的数据。

表达式可以使用二进制、逻辑、关系和其他类型的表达式组合，形成复杂或关系(match)表达式。

它们也被用作某些类型的操作的参数，如NAT、数据包标记等。

每个表达式都有一个数据类型，它决定符号值的数值长度、解析和表示以及表达式的类型兼容性。



## DESCRIBE 命令

**describe**
*expression*

**describe** 命令显示有关表达式的结果类型及其数据类型的信息。

**describe 命令**

```bash
$ nft describe tcp flags
payload expression, datatype tcp_flag (TCP flag) (basetype bitmask, integer), 8 bits
predefined symbolic constants:
fin                           	0x01
syn                           	0x02
rst                           	0x04
psh                           	0x08
ack                           	0x10
urg                           	0x20
ecn                           	0x40
cwr                           	0x80
```



## 数据类型（DATA TYPES）

数据类型（ **data type** ）决定符号值的数值长度、解析和表示以及表达式的类型兼容性。存在许多全局数据类型，此外，一些表达式类型进一步定义了特定于表达式类型的数据类型。大多数数据类型有一个固定的长度，但是有些可能有一个动态的长度，例如字符串类型。

- 类型可以从低阶类型派生，举例： IPv4地址类型派生自整数类型，这意味着IPv4地址也可以指定为一个整数值。
- 在某些上下文（context）中(set和map定义)，需要显式地指定数据类型。每种类型都有一个用于此的名称。

### INTEGER TYPE（整型）

| 名称    | 关键字  | 由以下类型派生 |
| ------- | ------- | -------- |
| Integer | 整数变量 | - |

 **NTEGER TYPE** 用于数值。它可以被指定为十进制、十六进制或八进制。整数类型没有固定的大小，它的大小由所使用的表达式决定。

### BITMASK TYPE（位掩码类型）

| 名称    | 关键字                     | 由以下类型派生 |
| ------- | -------------------------- | --------- |
| Bitmask | bitmask 变量 | integer |

位掩码类型(bitmask)用于位掩码。

### STRING TYPE（字符串类型）

| 名称   | 关键字 | 由以下类型派生 |
| ------ | ------ | -------------- |
| String | string | -              |

**STRING TYPE** 用于字符串。字符串以字母字符(A- za -z)开始，后跟零个或多个字母数字字符或 /、-、_和 .. 此外，任何包含在双引号(")中的内容都被识别为字符串。

```
# Interface name
filter input iifname eth0
# Weird interface name
filter input iifname "(eth0)"
```



### LINK LAYER ADDRESS TYPE（链路层地址类型）

| 名称               | 关键字 | 数据长度 | 由以下类型派生 |
| ------------------ | ------ | -------- | -------------- |
| Link layer address | lladdr | variable | integer        |

**LINK LAYER ADDRESS TYPE **用于链路层地址。链路层地址被指定为由两个用冒号分隔的十六进制数字组成的可变数量的组。

**Link layer address 规范**

```
# Ethernet destination MAC address
filter input ether daddr 20:c9:d0:43:12:d9
```



### IPV4 ADDRESS TYPE（IPv4 地址类型）

| 名称      | 关键字 | 数据长度 | 由以下类型派生 |
| --------- | ------ | -------- | -------------- |
| IPv4 地址 | lladdr | 32 位    | integer        |

**IPV4 ADDRESS TYPE** 用于IPv4地址。地址可以指定为点分（隔）十进制（192.168.1.1）、点分十六进制（c0.a8.1.1）、点分八进制、十进制(12625937)、十六进制(0xc0a811)、八进制，也可以指定为主机名。主机名将使用标准系统解析器进行解析。

**IPv4 地址规范**

```
# 使用点分隔的十进制数字
filter output ip daddr 127.0.0.1
# 使用主机名
filter output ip daddr localhost
```



### IPV6 ADDRESS TYPE（IPV6 地址类型）

| 名称      | 关键字    | 数据长度 | 由以下类型派生 |
| --------- | --------- | -------- | -------------- |
| IPv6 地址 | ipv6_addr | 128  位  | integer        |

**IPV6 ADDRESS TYPE** 用于IPv6地址。地址以主机名或以冒号分隔的十六进制半字指定。地址可以用方括号(“[]”)括起来，以区别于端口号。

**IPv6地址规范（含方括号使用规范）**

```
# 简短的回环地址(loopback)
filter output ip6 daddr ::1

# 如果没有[ ]，端口号(22)将被解析为ipv6地址的一部分
ip6 nat prerouting tcp dport 2222 dnat to [1ce::d0]:22
```



### BOOLEAN TYPE（布尔类型）

| 名称     | 关键字  | 数据长度 | 由以下类型派生 |
| -------- | ------- | -------- | -------------- |
| 布尔类型 | boolean | 1 位     | integer        |

布尔类型是用户空间中的语法辅助类型（syntactical helper type ）。它的用途是在关系表达式的右边(通常是隐式的)将左边的表达式更改为布尔检查(通常是为了存在，**existence** )。

以下关键字将自动解析为给定值的布尔类型：

| 关键字  | 值   |
| ------- | ---- |
| exists  | 1    |
| missing | 0    |

**Boolean 规范**

下面的表达式支持布尔比较：

| 表达式     | 行为                    |
| ---------- | ----------------------- |
| fib        | 检查路由的存在。        |
| exthdr     | 检查IPv6扩展头的存在。  |
| tcp option | 检查TCP选项报头的存在。 |



```
# 如果路由存在，则匹配（match）
filter input fib daddr . iif oif exists
# 只匹配IPv6流量中的非分片报文
filter input exthdr frag missing
# 如果TCP时间戳选项是存在的,则匹配
filter input tcp option timestamp exists
```



### ICMP TYPE TYPE（ICMP TYPE 类型）

| 名称      | 关键字    | 数据长度 | 由以下类型派生 |
| --------- | --------- | -------- | -------------- |
| ICMP Type | icmp_type | 8 位     | integer        |

**ICMP TYPE**类型用于方便地指定ICMP报头的类型字段。

在指定ICMP类型时，可以使用以下关键字:

| 关键字                  | 数值 |
| ----------------------- | ---- |
| echo-reply              | 0    |
| destination-unreachable | 3    |
| source-quench           | 4    |
| redirect                | 5    |
| echo-request            | 8    |
| router-advertisement    | 9    |
| router-solicitation     | 10   |
| time-exceeded           | 11   |
| parameter-problem       | 12   |
| timestamp-request       | 13   |
| timestamp-reply         | 14   |
| info-request            | 15   |
| info-reply              | 16   |
| address-mask-request    | 17   |
| address-mask-reply      | 18   |

**ICMP Type 规范**

```
# 匹配 ping 包，包括两种类型的echo-request及echo-reply
filter output icmp type { echo-request, echo-reply }
```



### ICMP CODE TYPE（ICMP CODE类型）

| 名称      | 关键字    | 数据长度 | 由以下类型派生 |
| --------- | --------- | -------- | -------------- |
| ICMP Code | icmp_code | 8 位     | integer        |

ICMP Code 类型用于方便地指定ICMP头的Code字段。

在指定 *ICMP code* 时，可以使用以下关键字:

| 关键字           | 数值 |
| ---------------- | ---- |
| net-unreachable  | 0    |
| host-unreachable | 1    |
| prot-unreachable | 2    |
| port-unreachable | 3    |
| net-prohibited   | 9    |
| host-prohibited  | 10   |
| admin-prohibited | 13   |



### ICMPV6 TYPE TYPE

| 名称        | 关键字      | 数据长度 | 由以下类型派生 |
| ----------- | ----------- | -------- | -------------- |
| ICMPv6 Type | icmpv6_type | 8 位     | integer        |

ICMPV6 TYPE 类型用于方便地指定ICMPv6报头的 **type** 字段。

当指定 **ICMPv6 type** 时，可以使用以下关键字:

| 关键字                  | 数值 |
| ----------------------- | ---- |
| destination-unreachable | 1    |
| packet-too-big          | 2    |
| time-exceeded           | 3    |
| parameter-problem       | 4    |
| echo-request            | 128  |
| echo-reply              | 129  |
| mld-listener-query      | 130  |
| mld-listener-report     | 131  |
| mld-listener-done       | 132  |
| mld-listener-reduction  | 132  |
| nd-router-solicit       | 133  |
| nd-router-advert        | 134  |
| nd-neighbor-solicit     | 135  |
| nd-neighbor-advert      | 136  |
| nd-redirect             | 137  |
| router-renumbering      | 138  |
| ind-neighbor-solicit    | 141  |
| ind-neighbor-advert     | 142  |
| mld2-listener-report    | 143  |

**ICMPv6 Type 规范**

```
# 匹配 ICMPv6 协议的ping包
filter output icmpv6 type { echo-request, echo-reply }
```



### ICMPV6 CODE TYPE（ICMPV6 CODE 类型）

| 名称        | 关键字      | 数据长度 | 由以下类型派生 |
| ----------- | ----------- | -------- | -------------- |
| ICMPv6 Code | icmpv6_code | 8 位     | integer        |

 ICMPv6 Code 类型用于方便地指定ICMPv6报头的 **code** 字段。

指定 **ICMPv6 code** 时可以使用以下关键字：

| 关键字           | 数值 |
| ---------------- | ---- |
| no-route         | 0    |
| admin-prohibited | 1    |
| addr-unreachable | 3    |
| port-unreachable | 4    |
| policy-fail      | 5    |
| reject-route     | 6    |



### ICMPVX CODE TYPE（ICMPVX CODE 类型）

| 名称        | 关键字     | 数据长度 | 由以下类型派生 |
| ----------- | ---------- | -------- | -------------- |
| ICMPvX Code | icmpx_code | 8 位     | integer        |

ICMPvX Code 类型抽象是一组值，它重叠在inet家族中使用的ICMP和ICMPv6 Code类型之间。

ICMPvX Code 类型是抽象的一组值，在 **inet ** 簇中使用的ICMP和ICMPv6 Code类型之间的重叠部分。

当指定 **ICMPvX code** 时，可以使用以下关键字：

| 关键字           | 数值 |
| ---------------- | ---- |
| no-route         | 0    |
| port-unreachable | 1    |
| host-unreachable | 2    |
| admin-prohibited | 3    |



### CONNTRACK TYPES（CONNTRACK 类型）

这是 **ct** 表达式和语句中使用的类型的概述:

| 名称                 | 关键字    | 数据长度 | 由以下类型派生 |
| -------------------- | --------- | -------- | -------------- |
| conntrack state      | ct_state  | 4 byte   | bitmask        |
| conntrack direction  | ct_dir    | 8 bit    | integer        |
| conntrack status     | ct_status | 4 byte   | bitmask        |
| conntrack event bits | ct_event  | 4 byte   | bitmask        |
| conntrack label      | ct_label  | 128 bit  | bitmask        |

为了方便，以上每种类型都提供了关键词:

**conntrack state (ct_state)**

| Keyword     | Value |
| ----------- | ----- |
| invalid     | 1     |
| established | 2     |
| related     | 4     |
| new         | 8     |
| untracked   | 64    |

**conntrack direction (ct_dir)**

| Keyword  | Value |
| -------- | ----- |
| original | 0     |
| reply    | 1     |

**conntrack status (ct_status)**

| Keyword    | Value |
| ---------- | ----- |
| expected   | 1     |
| seen-reply | 2     |
| assured    | 4     |
| confirmed  | 8     |
| snat       | 16    |
| dnat       | 32    |
| dying      | 512   |

**conntrack event bits (ct_event)**

| Keyword   | Value |
| --------- | ----- |
| new       | 1     |
| related   | 2     |
| destroy   | 4     |
| reply     | 8     |
| assured   | 16    |
| protoinfo | 32    |
| helper    | 64    |
| mark      | 128   |
| seqadj    | 256   |
| secmark   | 512   |
| label     | 1024  |

**conntrack label** 类型( *ct_label* )可能的关键字在运行时从 /etc/connlabel.conf 中读取。



## 主表达式（PRIMARY EXPRESSIONS）

最低顺序的表达式是一个主表达式，表示一个常量或来自包的有效负载（payload）、元数据（meta data）或有状态模块(stateful module)的单个数据。



### META 表达式 （元数据查询）

**meta** {length | nfproto | l4proto | protocol | priority}

[meta] {mark | iif | iifname | iiftype | oif | oifname | oiftype | skuid | skgid | nftrace | rtclassid | ibrname | obrname | pkttype | cpu | iifgroup | oifgroup | cgroup | random | secpath}

**meta** 表达式指的是与数据包相关联的元数据（meta data）。

元表达式有两种类型：非限定元表达式和限定元表达式。限定的元表达式要求在元键之前使用 **meta** 关键字。限定元表达式可以通过直接使用元键或作为限定元表达式来指定。14proto元键对于匹配IPv4或IPv6数据包的特定传输协议非常有用。它还将跳过IPv6数据包中存在的任何IPv6扩展头。

元表达式类型：

| Keyword   | Description                                          | Type              |
| --------- | ---------------------------------------------------- | ----------------- |
| length    | 数据包的长度，以字节为单位                           | integer (32 bit)  |
| nfproto   | real hook protocol family, useful only in inet table | integer (32 bit)  |
| l4proto   | 第4层协议相关，跳过ipv6扩展头                        | integer (8 bit)   |
| protocol  | EtherType 协议数值                                   | ether_type        |
| priority  | TC数据包优先级                                       | tc_handle         |
| mark      | 数据包标记（mark）                                   | mark              |
| iif       | 往内核流入数据包的网络接口索引（输入接口索引）       | iface_index       |
| iifname   | 往内核流入数据包的网络接口名称（输入接口名称）       | ifname            |
| iiftype   | 往内核流入数据包的网络接口的类型（输入接口类型）     | iface_type        |
| oif       | 从内核流出数据包的网路接口索引（输出接口索引）       | iface_index       |
| oifname   | 从内核流出数据包的网络接口名称（输出接口名称）       | ifname            |
| oiftype   | 从内核流出数据包的网络接口的类型（输出接口硬件类型） | iface_type        |
| skuid     | 与原始套接字关联的UID                                | uid               |
| skgid     | 与原始套接字相关的GID                                | gid               |
| rtclassid | 域路由表                                             | realm             |
| ibrname   | 输入桥接接口名称                                     | ifname            |
| obrname   | 输出桥接接口名称                                     | ifname            |
| pkttype   | 数据包类型                                           | pkt_type          |
| cpu       | 处理数据包的CPU编号                                  | integer (32 bits) |
| iifgroup  | incoming 设备组                                      | devgroup          |
| oifgroup  | outgoing  设备组                                     | devgroup          |
| cgroup    | control group id（cgroup子系统相关）                 | integer (32 bits) |
| random    | 伪随机数                                             | integer (32 bits) |
| secpath   | 布尔值                                               | boolean (1 bit)   |

**Meta 表达式特定类型**

| 类型          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| iface_index   | 网络接口索引 (32 位长度数字)。可以用数字或现有接口的名称指定（如eth0）。 |
| ifname        | 网络接口名称 (16 字节长度字符串)。该名称的接口不一定要存在。 |
| iface_type    | 接口类型 (16 位长度数字)。                                   |
| uid           | 用户标识 (32 位长度数字)。可以使用数字或用户名指定。         |
| gid           | 用于组标识 (32 位长度数字)。可以使用数字或用户组名指定。     |
| realm         | 域路由表 (32位长度数字). 可以使用数字或在 /etc/iproute2/rt_realms 中定义的符号名指定。 |
| devgroup_type | 设备组 (32 位长度数字). 可以使用数字或在 /etc/iproute2/group 中定义的符号名指定。 |
| pkt_type      | 数据包类型: Unicast (单播), Broadcast (广播), Multicast (组播). |

**使用元表达式**

```
# 限定元表达式
filter output meta oif eth0

# 非限定元表达式
filter output oif eth0

# 包装受ipsec处理的影响
raw prerouting meta secpath exists accept
```



### FIB 表达式（转发信息库查询）

**fib** {saddr | daddr | {mark | iif | oif}} {oif | oifname | type}

fib表达式通过查询fib( forwarding information base，转发信息库)来获取特定地址将使用的输出接口索引等信息。输入是一个元素元组，用作fib查找函数的输入。

**fib 表达式特定的类型**

| 关键字  | 描述         | 类型            |
| ------- | ------------ | --------------- |
| oif     | 输出接口索引 | integer (32 位) |
| oifname | 输出接口名称 | string          |
| type    | 地址类型     | fib_addrtype    |

**fib 表达式的使用**

```
# 在没有反向路径的情况下丢弃数据包
filter prerouting fib saddr . iif oif missing drop

# 丢弃未配置入接口地址的报文
filter prerouting fib daddr . iif type != { local, broadcast, multicast } drop

# 在特定的“blackhole”表中执行查找(0xdead，需要适当的IP规则)
filter prerouting meta mark set 0xdead fib daddr . mark type vmap { blackhole : drop, prohibit : jump prohibited, unreachable : drop }
```



### ROUTING 表达式 （路由查询）

**rt** {classid | nexthop}

路由表达式指的是与数据包相关联的路由数据。

**路由表达式类型**

| Keyword | Description            | Type                |
| ------- | ---------------------- | ------------------- |
| classid | 域路由表               | realm               |
| nexthop | 路由下一跳地址         | ipv4_addr/ipv6_addr |
| mtu     | 路由的 TCP最大报文长度 | integer (16 bit)    |

**路由表达式特定类型**

| 类型  | 描述                                                         |
| ----- | ------------------------------------------------------------ |
| realm | 域路由表 (32 bit number). 可以用数字或 /etc/iproute2/rt_realms 中定义的符号名称来指定。 |

**路由表达式的使用**

```
# ip地址簇独立的rt表达式
filter output rt classid 10

# 依赖ip簇的rt表达式
ip filter output rt nexthop 192.168.0.1
ip6 filter output rt nexthop fd00::1
inet filter output rt ip nexthop 192.168.0.1
inet filter output rt ip6 nexthop fd00::1
```



### PAYLOAD 表达式 (有效载荷)

有效载荷表达式指的是数据包有效载荷中的数据。



#### ETHERNET HEADER 表达式 （以太网头部）

**ether** [*Ethernet header field*]

**Ethernet header** ：

| Keyword | Description             | Type       |
| ------- | ----------------------- | ---------- |
| daddr   | Destination MAC address | ether_addr |
| saddr   | Source MAC address      | ether_addr |
| type    | EtherType               | ether_type |



#### VLAN HEADER 表达式（虚拟局域网头部）

**vlan** [*VLAN header field*]

**VLAN header** ：

| Keyword | Description                | Type             |
| ------- | -------------------------- | ---------------- |
| id      | VLAN ID (VID)              | integer (12 bit) |
| cfi     | Canonical Format Indicator | integer (1 bit)  |
| pcp     | Priority code point        | integer (3 bit)  |
| type    | EtherType                  | ether_type       |

