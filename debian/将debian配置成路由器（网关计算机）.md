# debian 简易路由(网关计算机)笔记

目标，利用已有的软件包，多网卡配置一个路由器

还需要继续的配置学习：

- tc流量控制
- 多个wan口配置及均衡负载
- PPPOE拨号（学习了部分）

事实上我在github看到一个[关于pppoe集成到systemd的讨论](https://github.com/systemd/systemd/issues/481)，抄作业即可

---

# 路由能力
- NAT 动态地址转换

- LAN口可以DHCP动态分配IP地址，代理DNS服务。

- 防火墙，实现常规的流量过滤功能。

- 可以设置端口映射

# 硬件

- 主机一台

- 网卡2口，一口做wan，一口做lan

# 涉及软件包

**系统**：debian10

**防火墙**：nftables（代替iptables）

**DHCP/DNS服务**：dnsmasq

**网卡接口管理服务**：systemd-udevd systemd-networkd

**流量控制和策略路由**：iproute2 

---

# 配置思路

- （1）使用systemd.link配置网卡命名规则，设置固定接口名方便修改 **nftables** 规则

- （2）使用 networkctl 配合 system-networkd 管理网卡

- （3）使用 nftables 配置 nat 规则以及防火墙规则

- （4）使用 dnsmasq 绑定 lan 口网卡，启用DHCP服务/DNS服务


## 安装软件包

```bash
sudo apt install nftables dnsmasq iproute2
```

## 新建`systemd.link`规则修改网卡名

步骤流程：

- （1） 卸载掉现在的网络管理（主要针对`networking`和`NetworkManager`）

- （2） 修改udev规则，达到修改网卡名的目的（防火墙规则需要判断网卡名）

- （3） 编写防火墙规则，使得内网流量出去时会被做SNAT。

- （4） 创建DHCP服务、DNS服务，为局域网下的主机分配动态分配IP。

### 1.通过`ip addr` 查询 网卡MAC地址

例如

```bash
#ip addr
2: enp4s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 23:33:33:33:33:04 brd ff:ff:ff:ff:ff:ff
    
3: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 23:33:33:33:33:05 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.33/24 brd 192.168.31.255 scope global dynamic 
```
**link/ehter** 项目对应的是 mac地址，记住他们

- **wan网络接口：** `23:33:33:33:33:04`

- **lan网络接口：** `23:33:33:33:33:05`

> 后续有用到这两个MAC地址的配置根据实际情况替换。


### 2.通过mac地址匹配udev规则来修改网卡名

这一小节需要参考的文档有[systemd.link 中文手册](http://www.jinbuguo.com/systemd/systemd.link.html#),利用mac地址匹配网卡设备，更改网卡名。


使用systemd.link为wan、lan网口创建网络接口名称命名规则的配置，创建配置文件：

```bash
sudo touch /lib/systemd/network/01-wan.link
sudo touch /lib/systemd/network/01-lan.link
```

文件内容如下，主要通过MAC地址进行匹配，参数含义参考 **systemd.link** 手册，link配置由 **systemd-udevd** 读取。

```ini
#/lib/systemd/network/01-wan.link
[Match]
MACAddress=23:33:33:33:33:04

[Link]
Description=wan interface
Name=wan


#/lib/systemd/network/01-lan.link
[Match]
MACAddress=23:33:33:33:33:05

[Link]
Description=lan interface
Name=lan  # 新的接口名称

```


## 切换`systemd-networkd`接管网卡，卸载networking和NetWorkManager服务

- 使用`ifupdown`包做网络管理，对应的服务为`networking.service`

- 使用`network-manager`对应的则是`NetworkManager.service`

网络管理软件可能通过两个源头来触发网络配置：服务与udev规则；如果禁用服务失败，需要考虑是否因为udev的rule触发事件导致配置变化。


### 1.卸载其他网络管理

```bash
sudo apt purge ifupdown network-manager
```

网络管理服务一般情况只存在一个，此处使用`systemd-networkd`作为网络管理服务，它包括:

- **systemd-resolved.service**（DNS服务）

- **systemd-networkd.service**（网络管理服务）

- **systemd-networkd-wait-online.service**（等待网络在线服务，用于阻塞）

三个相关服务，对应的命令有`networkctl`、`resolvectl`。





### 2.编写systemd.network配置文件

创建 **10-wan.network** 、**10-lan.network** 两个配置,文件名没要求，后缀要求是  **.network**

```
sudo touch /etc/systemd/network/10-wan.network
sudo touch /etc/systemd/network/10-lan.network
```

### 3.编辑规则

> 注：wan口暂时没有学习配置PPPOE拨号，还得继续学习。

wan口暂时设置为DHCP方式上网

wan口网络配置文件内容如下：

```ini
#/etc/systemd/network/10-wan.network

[Match]
MACAddress=23:33:33:33:33:04

[Link]
RequiredForOnline=no

[Network]
Description=wan network
DHCP=yes
IPForward=yes
NTP=ntp.aliyun.com
```

lan口网络配置文件内容如下：

```ini
# /etc/systemd/network/lan.network

# lan口静态ip 192.168.31.1
[Match]
MACAddress=23:33:33:33:33:05

[Link]
RequiredForOnline=no

[Network]
Description=LAN network
Address=192.168.31.1/24
IPForward=yes
```

### 4.启用systemd-networkd服务,允许开机自启

```bash
sudo systemctl daemon-reload
sudo systemctl enable systemd-networkd
sudo systemctl start systemd-networkd
```

### 5.使用systemd-resolved.service管理主机DNS,配置关闭监听53端口及5355端口

使用systemd-networkd管理网络，还需要启用systemd-resolved管理本机的DNS服务，确保域名解正常。

编辑`systemd-resolved.service` 配置文件如下：

```ini
# /etc/systemd/resolved.conf

[Resolve]
DNS=223.5.5.5 223.6.6.6
Cache=yes
DNSStubListener=no
ReadEtcHosts=yes
LLMNR=no
```

选项配置如下：

- （1） DNS: 空格分隔的上级DNS服务器，这里使用阿里云dns

- （2） Cache：设置缓存解释成功的域名

- （3） DNSStubListener：禁用本地DNS服务器，这个选项设置为no避免监听127.0.0.1:53

- （4） ReadEtcHosts：设置为yes,发送查询请求前，会优先查询/etc/hosts

- （5） LLMNR：设置为no，它将不会监听5355端口。

启动并允许开机启动：

```bash
sudo systemctl enable systemd-resolved.service
sudo systemctl start systemd-resolved.service
```

由于使用了 **systemd-resolved** ，dns更新与传统的保持一致，让 **/etc/resolv.conf -> /run/systemd/resolve/resolv.conf** ，保证更新一致。

```bash
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
```



## sysctl调整参数

对于内核网络调整方面接触较少，本节网上抄作业，但是调整参数选项的含义需要参考[linux内核网络文档](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/networking/ip-sysctl.rst)。

主要的调整目标为：

- 允许流量转发

- 不允许外网ping内网的IP地址

    

    在linux下，IP并不是绑定网卡的，可能出现这样的情况，A网卡收到一个arp查询，查询的目的IP是B网卡的，但是它可能响应A网卡的MAC地址，而不是B网卡的。我不想出现，外网企图询问lan口ip，也会获得一个应答。下面的调整，将检查查询请求的源地址，它查询的目的IP，不符合规则，则不做出响应。

添加以下配置：

```bash
# /etc/sysctl.conf

net.ipv4.tcp_syncookies=1
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.default.arp_ignore = 1
net.ipv4.conf.default.rp_filter = 1
```

调整说明：

- accept_source_route = 0 不接受路由报头。
- arp_announce = 2 定义接口上发送的ARP请求中的IP报文宣布本端源IP地址的不同限制级别，当值为2时，总是为这个目标使用最佳的本地地址。在这种模式下，忽略IP包中的源地址，并尝试选择我们喜欢的本地地址与目标主机进行对话。通过在包含目标IP地址的出接口的所有子网中查找主IP地址来选择这种本地地址。如果没有找到合适的本地地址，我们就选择出接口或所有其他接口上的第一个本地地址，希望能够收到对我们请求的回应，甚至有时不管我们宣布的源IP地址是什么。
- arp_ignore = 1 对于收到的解析本端目标IP地址的ARP请求，定义不同的应答方式，值为1时，只有当目标IP地址为入接口上配置的本端地址时才进行应答。
-  rp_filter = 1 严格模式RFC3704中定义的严格反向路径对每个入方向的报文进行FIB测试，如果接口不是最佳反向路径则报文检查失败。缺省情况下，失败的报文将被丢弃。





## 编写防火墙规则

### 1.编辑`/etc/nftables.conf`,写入具体规则

- 默认外网从wan口进来的流量访问的端口是被过滤的。

- 允许从内网主动发起的链接通过。

- 允许局域网内访问网关计算机。

- 网关计算机允许被ping。

- 从lan口流入内核，又没从lan口流出的流量，并且源ip符合ip段的流量，在postrouting钩子下的链要做masquerade规则，做特殊的SNAT，在本路由器处，做源地址转换，替换成`wan`口的网络地址，当数据包返回的时候，将目的地址替换成远程访问的IP。

> 此处的规则有个坑，因为 `input` 钩子下的规则，默认的行为是把阻止链接，如果网关计算机要监听某个端口，需要在 `/etc/nftables.conf` 里添加放行规则。

 `/etc/nftables.conf` 文件内容（comment关键字可以为规则添加注释，增强可读性）

```bash
flush ruleset

# IPv4协议簇的表，用于储存nat相关的规则，此处只用到prerouting和postrouting的钩子
# 这里有个问题，如果{后的空行，因为对齐的原因被补全了个tab，可能出现语法错误，如果单独空行，不应该用任何的空白符号占用它，但是可以用回车换行，单行即无注释又无定义语句，且有空白字符占用空行，可能出现多余的字符造成错误
table ip nat {

        chain prerouting-public {
            type nat hook prerouting priority 100; policy accept
            #如果需要端口转发，则在PREROUTING钩子的链做DNAT
            iif wan tcp dport 65533 dnat to 192.168.31.2:22 comment "dnat: :65533 => 192.168.31.2:22"
            # 把wan口进来的流量，目标端口是65533的发往192.168.31.2的22端口
        }

        chain postrouting-public {
                type nat hook postrouting priority 100; policy accept;
                #做动态SNAT （备注②）
                meta iif lan oif != lan ip saddr 192.168.31.0/24 masquerade comment "lan口流量外网NAT规则"
        }
}


# IPv4防火墙规律规则，主要针对进本机的流量
table ip filter {
        chain input-public {
                #默认的策略是不允许流量通过
                type filter hook input priority 0; policy drop;

                iif {lo,lan} accept comment "允许lo口、lan口流进的内网流量通过"
                icmp type echo-request counter accept  comment "允许被ping"
                #备注①
                ct state established,related accept comment "允许从内部主动发起的链接通过"
        }
}


# IPv6防火墙规律规则，主要针对进本机的流量
table ip6 filter {
        chain input-public {
                type filter hook input priority 0; policy drop
                ct state established,related accept comment "允许从内部主动发起的链接通过"
                iif {lo, lan} accept comment "允许lo口、lan口流进的内网流量通过"
                icmpv6 type {echo-request,nd-neighbor-solicit} accept comment "允许被ping"
        }
}
```

> 备注①：**ct state established,related accept;** 这条规则匹配链接状态的，链接状态和iptables的保持一致，有4种状态，**ESTABLISHED**、**NEW**、**RELATED**及**INVALID**，链接过程，客户端发出请求，服务端返回结果，刚好源地址/目的地址是相反的，第一个穿越防火墙的数据包，链接状态是**NEW**，后续的数据包（无论是请求还是应答）链接状态是**ESTABLISHED**，有一种情况，类似于FTP，区分数据端口和控制端口的，被动产生的数据包，第一个，但是不属于任何链接中的，它的状态是**RELATED**，而这个链接产生后，这条链路后续的数据包状态都是**ESTABLISHED**，最后一种是三种状态之外的数据包。


具体的情况分析：

- （1） 由本地作为客户端，主动去访问服务端（就好像我们访问网站的情况），此时我们的请求，发送出去，我们第一个数据包是NEW状态,netfilter追踪这个链接的状态。

- （2） 请求发出后，这条链路下一个返回的数据包，已经是**ESTABLISHED**状态了，但是如果此时防火墙，没有允许相关状态的链接通过，默认策略是drop的，会导致我们的包可以发出去，但是回来时被input默认DROP规则过滤。

- （3） 允许**ESTABLISHED**状态的数据包通过，可以让本机主动发出的请求后，回来的数据包，可以通过防火墙。

> 备注②：masquerade 是一种特殊的SNAT,按nftables的wiki说明，它只有在postrouting的钩子下才有意义的，此时，已经选中了路由，准备要发往对应的网卡了，masquerade会把源ip替换成该网卡对应的ip，然后记录这条信息，当数据返回的时候（prerouting），它会查找记录，又会把masquerade前的源ip替换成目的的IP（DNAT），这样，相当于“隐藏”了网关计算机自身，把链路转发到masquerade前的源ip主机。

nftables 不像 iptables 有内置的链，它是没有预设的链的，链的名称可以用户自定义，但是nftables兼容iptables的配置，如果使用docker的时候，就容易出现一个问题，docker会改动防火墙规则，同时它会清空我的链，所以，我应该与内置链的名称不一致，保证它不会被清空.


## DNS/DHCP服务配置

使用`dnsmasq`守护进程作为网关计算机的DNS以及DHCP服务。

> /etc/systemd/resolved.conf的默认设置会占用了53端口和5355端口，需要额外设置，详情看前面。

### 1.安装

```
sudo apt install dnsmasq
```


### 2.配置

把 `dnsmasq`默认的配置都不要了,把sysV启动的软连接全部删了，把 `dnsmasq.service` 的服务也删除了。可以满足多个网卡配置dnsmasq，不应该使用包内自带的服务配置，由自己编辑一个适合情况的配置文件。

```sh
#删除sysV init 开机自启的软连接
find /etc/rc[0-9S].d -name "*dnsmasq*" -exec sudo rm {} \;
#删除dnsmasq默认包的服务配置
sudo rm sudo rm /lib/systemd/system/dnsmasq.service
#重载配置
sudo systemctl daemon-reload
```

编写新的dnsmasq服务的service模板，以适应多个网卡单独启动不同的dnsmasq服务。

```sh
sudo touch /etc/systemd/system/dnsmasq@.service
```

编写 `/etc/systemd/system/dnsmasq@.service` 内容：
```ini
# /etc/systemd/system/dnsmasq@.service

[Unit]
Description=IPv4 DHCP server on %I
Wants=network.target
After=network.target

[Service]
Type=forking
PIDFile=/run/dnsmasq@%I.pid
ExecStart=/usr/sbin/dnsmasq --except-interface=lo --pid-file=/run/dnsmasq@%I.pid --log-facility=/var/log/dnsmasq/%I.log   --interface=%I --conf-file=/opt/router-config/dnsmasq/%I/dnsmasq.conf  --dhcp-leasefile=/opt/router-config/dnsmasq/%I/dnsmasq.leases --dhcp-hostsfile=/opt/router-config/dnsmasq/%I/hosts.d/ --resolv-file=/opt/router-config/dnsmasq/%I/resolv.conf --conf-dir=/opt/router-config/dnsmasq/%I/conf.d
KillSignal=SIGTERM
KillMode=control-group
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target

```

 `%I` 是模板实例化后，被替换的名称，例如，`dnsmasq@lan.service` ， `%I` 就会被替换成 `lan`


软连接新建一个实例，这个服务是针对 `lan` 网卡的。创建 `dnsmasq` 配置存放的文件夹，我放在 `/opt/router-config/dnsmasq/lan/` 下，

```bash
# 针对lan创建dnsmasq服务软连接，实例化模板
sudo ln -s /etc/systemd/system/dnsmasq@.service /etc/systemd/system/dnsmasq@lan.service
# 针对lan创建网卡的配置文件夹
sudo mkdir -p /opt/router-config/dnsmasq/lan/
# 创建dnsmasq配置文件
sudo touch /opt/router-config/dnsmasq/lan/dnsmasq.conf
# 创建租约记录文件（分配的客户端，都会被记录到这）
sudo touch /opt/router-config/dnsmasq/lan/dnsmasq.leases
# 创建静态分配的IP主机配置（dhcp-host）
sudo mkdir -p /opt/router-config/dnsmasq/lan/
# 创建额外配置目录conf.d
sudo mkdir -p /opt/router-config/dnsmasq/lan/conf.d
# 创建自定义的resolv.conf文件
sudo touch /opt/router-config/dnsmasq/lan/resolv.conf
# 创建日志存放的文件夹
sudo mkdir -p /var/log/dnsmasq
# 创建pid文件
sudo touch /run/dnsmasq@lan.pid 

```

 -  `/var/log/dnsmasq` 为日志文件夹，文件夹下，`网卡名.log` 为不同服务实例独立的日志。例如 `lan.log`

 - `/run/dnsmasq@lan.pid` 为守护进程的pid记录文件。

 -  `/opt/router-config/dnsmasq/lan/` 目录结构如下：

```
lan/
├── conf.d
│   └── anti-ad-for-dnsmasq.conf
├── dnsmasq.conf
├── dnsmasq.leases
├── hosts.d
│   └── phone.ip
└── resolv.conf

```

 - `hosts.d` 文件夹记录了绑定mac分配IP的配置。

 -  `conf.d` dnsmasq服务的补充配置，这里用来设定屏蔽广告域名。（[anti-ad域名屏蔽列表](https://anti-ad.net/)）



配置说明：
```bash
--except-interface 排除监听的网卡，此处设定不监听lo（127.0.0.1）。

--pid-file 守护进程的pid文件路径，此处systemd.service需要通过pid文件确定主进程的PID,精确控制停止进程。

--log-facility 服务日志保存路径

--interface dnsmasq服务绑定的网卡名，模板用%I绑定

--conf-file 配置文件的路径

--dhcp-leasefile 租约记录文件的路径，这个文件由dnsmasq自己管理，我们不改动内容

--dhcp-hostsfile 针对特定的mac，分配ip绑定的主机配置，如果是文件夹，则读取文件夹下所有的文件，此处使用的是文件夹，一个文件对应一个主机。

--resolv-file 自定义resolv.conf路径，上游DNS服务器的配置文件，此处是阿里云的dns

--conf-dir 导入目的目录下的所有配置文件，可以用来导入额外的配置，例如屏蔽部分域名。
```

> `dnsmasq.conf` 配置文件格式，和命令行参数保持一致，但是不同的地方是，没有 `--` 前缀，例如 `port = 53` 等价命令行 `--port=53`,一行一个参数，对于没有右值的，则直接填左值，例如 `log-dhcp` 等价命令行参数 `--log-dhcp` 。其他的选项就可以根据manpage来配置，主要分成两部分，一部分配置dns服务器的，另外一部分则是关于DHCP的配置。此处只配置DHCP部分。

主配置文件内容：

```ini
# /opt/router-config/dnsmasq/lan/dnsmasq.conf

# DNS服务监听的端口，一般设置成53，绑定的地址为网卡的ip
port = 53
listen-address=192.168.31.1
# 缓存条目1000条
cache-size=10000

# DNS查询所有上游的服务器
all-servers
# 允许dbus总线
enable-dbus
# 本地hosts域名
bogus-priv

##########DHCP#####################
# DHCP 记录日志
log-dhcp
# dhcp选项 router为默认网关
dhcp-option=option:router, 192.168.31.1
# netmask 子网掩码
dhcp-option=option:netmask, 255.255.255.0
# domain
dhcp-option=option:domain-name, "my-router"
# dhcp分配给客户端的dns服务器
dhcp-option=option:dns-server, 192.168.31.1
# MTU值
dhcp-option=option:mtu, 1500

# 动态分配ip的范围，从2~254,租约时间为10小时
dhcp-range=192.168.31.2, 192.168.31.254, 10h
```

更多dhcp-option的选项可以通过`dnsmasq --help dhcp`查询到。

 `/opt/router-config/dnsmasq/lan/resolv.conf` 内容(它定义了上游查询服务器，最多只能使用2个)：

```ini
nameserver 223.5.5.5 223.6.6.6
```

 `MYCOMPUTER`内容，针对mac地址为 `23:33:33:33:33:44` 的主机固定分配IP为 `192.168.31.5`

 `/opt/router-config/dnsmasq/lan/hosts.d/MYCOMPUTER` 文件内容：

 ```ini
 23:33:33:33:33:44, 192.168.31.5
 ```

文件格式是 `--dhcp-host` 选项的右值，以上相当于 `--dhcp-host=23:33:33:33:33:44, 192.168.31.5` ，它可以是IPv4的，也可以是IPv6的，具体参考manpage。也是一行一个参数，不同的主机，可以分开不同的文件写配置，也会被读入。


文件配置完毕后，需要尝试启动 `dnsmasq@lan.service` ，并且允许它开机自启。

```bash
sudo systemctl daemon-reload
sudo systemctl start dnsmasq@lan.service

# 查看服务工作状况
sudo systemctl status dnsmasq@lan.service

# 允许开机自启
sudo systemctl enable dnsmasq@lan.service
```

---
# PPPoE方式上网

经过一段时间的努力，开始改造成PPPoE拨号上网，和DHCP方式上网不一样，PPPoE方式上网需要额外的配置

PPPoE需要的软件包

```bash
sudo apt install pppoe
```

pppoe包包含了一些启动脚本

```
/usr/sbin/pppoe
/usr/sbin/pppoe-connect
/usr/sbin/pppoe-relay
/usr/sbin/pppoe-server
/usr/sbin/pppoe-sniff
/usr/sbin/pppoe-start
/usr/sbin/pppoe-status
/usr/sbin/pppoe-stop
```

pppoe包依赖ppp包，而ppp包又包含了一些ppp协议相关支持



PPPoE方式上网需要改动wan口的配置，上述wan采用的是dhcp/静态IP上网的方式，pppoe上网可能需要解决一些问题

- ppp拨号
- 是否需要访问光猫管理接口

pppoe上网，wan口是不需要配置ip地址的，因为流量经过pppoe协议封装。我将做以下调整

配置 **netdev** ，将一个wan口扩展成两个虚拟的网络接口，一个用于ppp拨号，一个用于连接光猫的管理接口，注意他们的mac地址是不一样的。

配置 **.network** 设定wan口与其他虚拟网络接口的绑定关系，再设置每个接口的IP设置。

由于pppoe拨号的接口是不需要IP地址的，我们将DHCP关闭。

假设光猫的管理接口是192.168.1.1，虚拟接口与他同网段192.168.1.2的静态IP

```ini
# /etc/systemd/network/01-wan-modem.netdev
[NetDev]
Description=wan-modem虚拟网卡用于访问光猫
Name=wan-modem
Kind=macvtap
MACAddress=23:33:33:33:33:06

[MACVTAP]
Mode=private


# /etc/systemd/network/01-wan-ppp.netdev
[NetDev]
Description=wan-ppp 虚拟网卡用于PPPOE拨号
Name=wan-ppp
Kind=macvtap
MACAddress=23:33:33:33:33:07

[MACVTAP]
Mode=private


#  /etc/systemd/network/01-wan.network
[Match]
MACAddress=23:33:33:33:33:04

[Link]
RequiredForOnline=yes

[Network]
Description= wan interface
DHCP=no

# /etc/systemd/network/01-wan-ppp.network
[Match]
Name=wan-ppp

[Network]
DHCP=no

# /etc/systemd/network/01-wan-modem.network
[Match]
Name=wan-modem

[Network]
Address=192.168.1.2/24
```

更改拨号配置文件

```bash
# /etc/ppp/chap-secrets

# 设定宽带用户username 密码是passwd
"username" * "passwd"


# /etc/ppp/pppoe.conf

# 指定pppoe拨号的网络接口
ETH="wan-ppp"
# 指定拨号使用的上网账号
USER="username"

DEMAND=no
PEERDNS=no
DNSTYPE="NOCHANGE"
CONNECT_TIMEOUT=30
CONNECT_POLL=2
PING="."

# 指定PIDFILE
CF_BASE=`basename $CONFIG`
PIDFILE="/var/run/$CF_BASE-pppoe.pid"


SYNCHRONOUS=no
CLAMPMSS=1480
LCP_INTERVAL=30
LCP_FAILURE=3
PPPOE_TIMEOUT=100

FIREWALL=NONE
LINUX_PLUGIN="/lib/pppd/2.4.9/rp-pppoe.so"
```

由于配置了DNS不会被修改，pppoe不会获取并配置dns到本地，我需要配置一个静态的dns服务器，这里使用 `223.5.5.5` ，由于使用了



```
# /etc/systemd/system/pppoe.service
[Unit]
Description=PPPOE service
BindsTo=sys-subsystem-net-devices-wan.device
After=sys-subsystem-net-devices-wan.device


[Service]
Type=forking
ExecStart=/usr/sbin/pppoe-start
ExecReload=/usr/sbin/pppoe-stop;/usr/sbin/pppoe-start
ExecStop=-/usr/sbin/pppoe-stop
Restart=always

[Install]
WantedBy=multi-user.target
```

这个配置文件可能还是存在一些问题的，但是勉强能用，主要是让这个服务，要在wan接口启动后，才能启动它（我感觉在network-pre.target之后也行。。。），启动并允许开启自启

```bash
sudo systemctl enable pppoe
sudo systemctl start pppoe
```







---

TODO:

- IPv6分配，由于IPv4和IPv6的DHCP分配地址方式不一样，具体详细的选项，还需要琢磨下手册。

- 网卡启动和关闭时，通过udev规则触发服务重启

> 笔记最后更新时间：2021-04-02

---





# 参考文档

- [linux内核网络sysctl调参参考文档](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/networking/ip-sysctl.rst)

- [systemd中文手册](http://www.jinbuguo.com/systemd/systemd.index.html)

- - [systemd.link](http://www.jinbuguo.com/systemd/systemd.link.html#)

- - [systemd.service](http://www.jinbuguo.com/systemd/systemd.service.html#)

- - [systemd-networkd.service](http://www.jinbuguo.com/systemd/systemd-networkd.service.html#)

- - [systemd-udevd.service](http://www.jinbuguo.com/systemd/systemd-udevd.service.html#)
- - [systemctl](http://www.jinbuguo.com/systemd/systemctl.html#)


- [dnsmasq Manpage手册](http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html)

- [Nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page)

- - [nftables hooks概述](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks)

- - [nftables 10分钟参考](https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes)





