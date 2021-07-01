# Traffic shaping with a RHEL Router

流量整形（shaping）是将某些流量优先于其他流量的能力。网络是有限制的，当对网络的需求超过这些限制时，整形（shaping）使您能够选择什么流量最重要。通过整形，您可以选择延迟或丢弃（drop）较低优先级的流量，以确保更重要的流量畅通无阻。

虽然很容易理解整形（shaping）的需求，甚至它是如何工作的，但它的配置非常棘手。这项技术要求网络工程师理解排队规则（queuing disciplines）及它们如何影响网络。在Internet上下文中，它变得更加复杂，多个设备使用各自的队列系统移动数据包。

我们将简化复杂性，并研究适合大多数网络的特定流量整形配置。我们将对代码进行分解，并提供足够的解释，以确保您可以轻松地管理和修改它，以满足您的需要。

本文将开发并描述一个脚本，该脚本可以使Linux路由器在两个网络之间形成流量。它是为Red Hat Enterprise Linux (RHEL) 7.6版开发的，但它与任何现代Linux兼容。我们将假设以下两个网络配置。

```bash

terminal servers(WEST) <=>  RHEL ROUTER <=> computer(EAST)

```


WEST 和 EAST 是我们红帽路由器中连接两个网络的以太网设备的名称。东网的用户接入西网的终端服务器，同时接入其他网络服务。我们将了解如何将终端服务器流量(TCP端口3389)优先于所有其他流量。

# 如何在红帽中进行流量整形



红帽中的流量整形是通过 tc(8) 命令控制的，该命令是 iproute2 项目的一部分。tc(8) 使工程师能够执行广泛的流量控制操作，其中整形是其最常用的用途。

**qdisc** 是排队规则，它们定义了Linux系统的流量控制接口。它是数据包进入流量控制系统，然后离开到网络设备的地方。有很多 **qdisc** 供你选择，但我们将重点关注两个最重要的，tc-htb(8) 和 tc-sfq(8) ，稍后我们将对此进行解释。

**class** 是整形（ shaping ）发生的地方。类具有延迟（delay）、丢弃（drop）甚至重新排序（re-order）数据包的能力，是排队系统的核心。它们引入了一个层次结构，由父类（parent classes）流向子类（child classes）。正是这种结构赋予了工程师定义队列并将它们链接在一起的能力和灵活性。

**filter** 是您将包定向到 **class** 的方式。 **qdisc** 将定义紧随其后的默认路径，除非filter覆盖该路径。通过覆盖路径，工程师可以将数据包定向到一个 class，该类可以整形流量。filter提供了一个灵活的模式匹配系统来识别感兴趣的数据包，然后对它们进行分类。

## 警告

在处理流量整形时，需要注意两个重要的注意事项。首先，你只能塑造出站(例如出口)流量。为了解决这一限制并确保我们的优先级适用于所有网络流量，我们创建了两个队列，一个在我们的WEST网络接口上，一个在我们的EAST网络接口上。

第二个警告是当路由器连接到不同能力的网络时，我们如何管理速度。例如，我们假设路由器的WEST网络接口是一个20 Mbps(兆比特)的Internet连接，它的EAST网络接口是一个100 Mbps的LAN网络。我们必须将流量限制在最慢的链路上，否则更快的网络可能会压倒较慢的网络，而我们的塑形将没有任何作用。为了确保我们的塑形是强制的，我们将把WEST和EAST的速度限制在20 Mbps。

# 代码实现

下面是创建流量整形策略的bash脚本。

```bash
#!/bin/bash

configure() {
    local device=$1
    local maxrate=$2
    local limited=$3

    # Delete qdiscs, classes and filters
    tc qdisc del dev $device root 2> /dev/null

    # Root htb qdisc -- direct pkts to class 1:10 unless otherwise classified
    tc qdisc add dev $device root handle 1: htb default 10
    
    # Class 1:1 top of bandwith sharing tree
    # Class 1:10 -- rate-limited low priority queue
    # Class 1:20 -- maximum rate high priority queue
    tc class add dev $device parent 1: classid 1:1 htb rate $maxrate burst 20k
    tc class add dev $device parent 1:1 classid 1:10 htb \
        rate $limited ceil $maxrate burst 20k
    tc class add dev $device parent 1:1 classid 1:20 htb \
        rate $maxrate ceil $maxrate burst 20k

    # SFQ ensures equitable sharing between sessions
    tc qdisc add dev $device parent 1:10 handle 10: sfq perturb 10
    tc qdisc add dev $device parent 1:20 handle 20: sfq perturb 10

    # Classify ICMP into high priority queue
    tc filter add dev $device parent 1:0 protocol ip prio 1 u32 \
        match ip protocol 1 0xff flowid 1:20

    # Classify TCP-ACK into high priority queue
    tc filter add dev $device parent 1: protocol ip prio 1 u32 \
        match ip protocol 6 0xff \
        match u8 0x05 0x0f at 0 \
        match u16 0x0000 0xffc0 at 2 \
        match u8 0x10 0xff at 33 \
        flowid 1:20
}

main() {
    # Enable rate reporting in the htb scheduler
    echo 1 > /sys/module/sch_htb/parameters/htb_rate_est

    configure WEST 20mbit 5mbit
    configure EAST 20mbit 5mbit

    # Classify terminal server traffic outbound to WEST
    tc filter add dev WEST protocol ip parent 1: prio 2 u32 \
        match ip dport 3389 0xffff flowid 1:20

    # Classify terminal server traffic outbound to EAST
    tc filter add dev EAST protocol ip parent 1: prio 2 u32 \
        match ip sport 3389 0xffff flowid 1:20
}

main "$@"
```
