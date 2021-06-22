# debian 配置 ulogd2 学习笔记

## 简介
ulogd2 用户态netfilter日志记录服务。他可以配合netfilter的规则把数据包日志转换成json、sqlite3/mysql、pcap等多种格式文件，方便分析或者用于其他的数据处理（审计、分析等）。

[ulogd2英文文档参考](http://www2.kangran.su/~nnz/pub/nf-doc/ulogd2/)


## 安装

```bash
#安装nftables
sudo apt install nftables
#安装ulogd2服务
sudo apt install ulogd2

# 下面的插件包是可选的，可以用来输出json、pcap、以及把日志导入数据库等
ulogd2-dbi
ulogd2-json
ulogd2-mysql
ulogd2-pcap
ulogd2-pgsql
ulogd2-sqlite3

```

## 配置

`ulogd2`需要配合`nftables`（背后是netfilter防火墙）使用，使用nft添加规则，netfilter抛出日志，然后ulogd2监听读取这些日志后，ulogd2再存为对应的日志文件（例如生成cap、或者把信息导入数据库），是否有日志生成，还需要数据包符合防火墙规则。

ulogd2默认配置文件位于：`/etc/ulogd.conf`，它是.ini风格的配置文件。主要分成2类节点，`[global]`是针对`ulogd2`主要服务的配置，另外一类节点，则是为了实例化插件的配置，名称可以自定义，但是在`[global]`节中，`stack=`选项来指定插件实例化的配置。


### [global] 节点

`[global]`节主要的选项参数有

> `logfile`、`loglevel`、`plugin`、`stack`

`logfile` 为ulogd2服务本身日志输出设置，它可能是：文件名（字符串）、syslog、stdout

`loglevel` 为日志等级，取值可能为1-8， 1=debug information, 3=informational messages, 5=noticable exceptional conditions, 7=error conditions, 8=fatal errors, program abort.

`plugin` 参数为插件.so文件的路径（字符串），plugin选项可以有多个，对应不同的插件，可以导入多个插件，
有两种插件，解释器插件和输出插件。解释器插件解析数据包，输出插件将解释的信息写入一些日志文件、数据库等，插件选项可以出现多次。

```
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_inppkt_NFLOG.so"
#plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_inppkt_ULOG.so"
#plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_inppkt_UNIXSOCK.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_inpflow_NFCT.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_IFINDEX.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_IP2STR.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_IP2BIN.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_IP2HBIN.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_PRINTPKT.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_HWHDR.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_PRINTFLOW.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_MARK.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_LOGEMU.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_SYSLOG.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_XML.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_SQLITE3.so"
#plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_GPRINT.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_NACCT.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_PCAP.so"
#plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_PGSQL.so"
#plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_MYSQL.so"
#plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_DBI.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_raw2packet_BASE.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_inpflow_NFACCT.so"
#plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_GRAPHITE.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_JSON.so"
```

默认全部插件都被注释掉，根据需要去掉`#`注释，查看插件的名称可以通过`sudo ulogd -i so文件路径`命令查询插件信息。

`stack` 参数可以定义多个日志栈，它包含多个插件的实例化对象，这个选项后面跟着一个插件实例列表，它将以输入插件开始，包含可选的过滤插件和输出插件结束。此选项可能出现多次。它大概这样的：

```ini
[global]

....其他配置选项...

stack=log2:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,mac2str1:HWHDR,json1:JSON
```

log2:NFLOG  log2是log2节的`插件配置节点`,针对`NFLOG`配置，NFLOG则是插件名称，意思是这个是创建一个以`[log2]`节为配置的NFLOG插件实例，stack固定以输入插件开始的，至少包含一个。

base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,mac2str1:HWHDR 这些则定义了一堆数据转换的插件，但是有些插件并没有在配置文件里找到对应的节...可能是内置的名称吧，这个没有详细研究，可以直接复制使用。

json1:JSON  最后这个日志栈（stack）则以JSON插件配置的输出，一般只有一个，要放在最后，但是如果同一个stack中，数据类型不一样，可能会出现问题，配置的时候，需要留意过滤插件（filter plugin）对日志的格式转换，同一个stack中，对应的键需要匹配。

### 插件配置节点

无论是输入(Input plugin)还是输出插件(Output plugin)。都可以针对性设置配置，节点的名称可以自定义。配置参考可以先查看插件信息。

```bash
#查看/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_JSON.so
sudo ulogd -i /usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_JSON.so
Name: JSON
Config options:
        Var: file (String, Default: /var/log/ulogd.json)
        Var: sync (Integer, Default: 0)
        Var: timestamp (Integer, Default: 1)
        Var: eventv1 (Integer, Default: 0)
        Var: device (String, Default: Netfilter)
        Var: boolean_label (Integer, Default: 0)
Input keys:
        No statically defined keys
Output keys:
        Output plugin, No keys

```

如图，`JSON`为该插件的名称，在`stack=`选项实例化插件时需要用到的插件名称。需要根据`Config options`提供的字段名称和类型配置节点的字段，例如以下是个`JSON`输出插件（Output plugin）的配置，输出日志到`/var/log/ulogd/myjson-output.json`

```ini
[myjson]
file = "/var/log/ulogd/myjson-output.json"
sync = 1 #同步写入
device = "我的设备名称" #这个字段会影响json日志中的设备名称字段

```


---

## 插件节参考

首先,插件节（section）的名称并不重要，因为它只是在stack指定实例化插件的时候用到节的名称，节名称可以根据自己需求更改，在stack里对应就行了。

不同的插件，有基本的配置选项，通过以下命令查看，不同的插件，可以根据`/etc/ulogd2.conf`里的`plugin=`选项配置找到.so的文件路径，直接复制便可，一般ulogd2已经给出了默认的配置，只是它把这些配置注释掉了。下面是JSON的配置

```
sudo ulogd -i /usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_JSON.so

Name: JSON
Config options:
        Var: file (String, Default: /var/log/ulogd.json)
        Var: sync (Integer, Default: 0)
        Var: timestamp (Integer, Default: 1)
        Var: eventv1 (Integer, Default: 0)
        Var: device (String, Default: Netfilter)
        Var: boolean_label (Integer, Default: 0)
Input keys:
        No statically defined keys
Output keys:
        Output plugin, No keys

```

`Config options` 中定了插件节可以定义的字段,例如我要输出json格式的日志到 `/var/log/ulogd/icmp.log` ，目的是记录icmp数据包的日志。

首先定义一个用于JSON插件实例化的配置节点 `json_icmp` ，配置输出日志保存在 `/var/log/ulogd/icmp.log` 

```ini
[json_icmp]
sync=1
file="/var/log/ulogd/icmp.log"
timestamp=1
```

定义一个用于 `NFLOG` 插件实例化的配置节 `nflog_icmp` ，用来配置过滤输入日志分组，分组号为8，这个分组需要在nft配置log规则时用到。

```ini
[nflog_icmp]
group = 8
```

在`[global]`节中新增一个`stack=`，创建一个日志栈（stack）。

```ini
[global]
...
stack=nflog_ssh:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,mac2str1:HWHDR,json_ssh:JSON

```

它的输入插件是`NFLOG`，输出插件是`JSON`。

`NFLOG` 插件的Output keys有：`raw.pkt`、`raw.mac` ... 等等。和iptables不一样，nftables使用的目标是NFLOG，而iptables使用的是ULOG，Debian 10使用的是nftables。

`JSON` 输出插件，没定义Input keys。


用到以下插件：

- `BASE` raw数据转换成多个字段的插件，其中包括`ip.saddr`、`ip.daddr`等等..具体可以通过查看这个插件的Output Keys得到。而它的Input keys，则是`raw.pkt`、`raw.pktlen`、`oob.family`、`oob.protocol`

- `IFINDEX` 把网络接口转换成字符串名称，例如`enp3s0`,在输入日志的时候，会显示具体名称的字符串。

- `IP2STR` 把IP数据转换成字符串,例如`192.168.1.1`,结果字符串。

- `HWHDR` 把MAC数据转换成字符串，例如`FF:FF:FF:FF:FF:FF`，结果是字符串。

这几个过滤插件（filter plugin），用于转换数据格式，但是这些插件不需要额外配置节。可以直接复制预设的配置使用。

根据需要用到的过滤插件，去掉注释。

```ini
[global]
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_inppkt_NFLOG.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_IFINDEX.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_IP2STR.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_filter_HWHDR.so"
plugin="/usr/lib/x86_64-linux-gnu/ulogd/ulogd_output_JSON.so"
```

修改配置文件后，重启`ulogd2`服务

```bash
sudo systemctl restart ulogd2
```

重启后可以查看启动后，plugin载入情况以及stack创建情况。

```bash
sudo systemctl status ulogd2
```

正常应该有类似这样的提示
```log
ulogd[74761]: registering plugin `NFLOG'
...

building new pluginstance stack: 'nflog_ssh:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,mac2str1:HWHDR,json_ssh:JSON'

```

当然可能会因为一些配置上的错误，导致出问题，例如数据库文件不存在，又或者因为Input keys 和Output Keys类型不匹配，可能有这样的提示

```
type mismatch between JSON and LOGEMU in stack
type mismatch between PCAP and JSON in stack
```

上面这种是在一个stack里用了多个输出插件（Output plugin）的情况。

配置完成后，还需要nftables创建规则


```bash
# 创建filter表
sudo nft 'add table ip filter'
# 创建INPUT链
sudo nft 'add chain ip filter INPUT {type filter hook input priority 0;}'
# 创建记录icmp包的规则
sudo nft 'add rule ip filter INPUT ip protocol icmp log group 8'
```

尝试去ping本主机，或者ping别人，ICMP协议的数据包信息就会被记录下来，保存在`/var/log/ulogd/icmp.log`

一行一个JSON结构的数据，按行读出即可。


## ulogd2 保存下来的数据有什么用？

可以用于网络分析，例如网络审计，但是毕竟只是根据数据包生成的日志或者是保存的cap文件。分析二三四层协议问题不大，但是对于再上层协议的分析（例如http），还需要找其他工具或者自己实现。


