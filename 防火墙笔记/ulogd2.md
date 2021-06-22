# ulogd2配置


## 简介
ulogd2 用户态netfilter日志记录服务。他可以配合netfilter的规则把数据包日志转换成json、sqlite3/mysql、pcap等格式文件，方便分析或者用于其他的数据处理（审计、分析等）。

## 安装

```
sudo apt install ulogd2 ulogd2-sqlite3 ulogd2-json
```

ulogd2相关的包有`ulogd2-xxx`，debian把这些模块细分成很多个包，上面根据需求下载了json和sqlite3的，其他相关可以通过下面命令搜索，根据需要下载

```
sudo apt search ^ulogd2
```


## 相关文件介绍

```bash
# ulogd2服务配置文件位置
/etc/ulogd.conf 

# ulogd2 文档位置
/usr/share/doc/ulogd2/

# sqlite3 创建表的sql语句(这两个文件是一样的)
/usr/share/doc/ulogd2/sqlite3.table
/usr/share/doc/ulogd2-sqlite3/sqlite3.table

```

`/proc/net/netfilter/nf_log` 中可以看到日志的分组

```
 0 NONE (nfnetlink_log)
 1 NONE (nfnetlink_log)
 2 nfnetlink_log (nfnetlink_log)
 3 NONE (nfnetlink_log)
 4 NONE (nfnetlink_log)
 5 NONE (nfnetlink_log)
 6 NONE (nfnetlink_log)
 7 nfnetlink_log (nfnetlink_log)
 8 NONE (nfnetlink_log)
 9 NONE (nfnetlink_log)
10 nfnetlink_log (nfnetlink_log)
11 NONE (nfnetlink_log)
12 NONE (nfnetlink_log)
```

这个分组是根据协议簇来的,留意上下对应
The syntax is the following FAMILY ACTIVE_MODULE (AVAILABLE_MODULES).

```
#define AF_UNSPEC	0
#define AF_UNIX		1	/* Unix domain sockets 		*/
#define AF_INET		2	/* Internet IP Protocol 	*/
#define AF_AX25		3	/* Amateur Radio AX.25 		*/
#define AF_IPX		4	/* Novell IPX 			*/
#define AF_APPLETALK	5	/* Appletalk DDP 		*/
#define	AF_NETROM	6	/* Amateur radio NetROM 	*/
#define AF_BRIDGE	7	/* Multiprotocol bridge 	*/
#define AF_AAL5		8	/* Reserved for Werner's ATM 	*/
#define AF_X25		9	/* Reserved for X.25 project 	*/
#define AF_INET6	10	/* IP version 6			*/
#define AF_MAX		12	/* For now.. */
```

组0是内核用来记录 非法的ct信息，所以，[log1]和[emu1]组的信息是不变的

Group O is used by the kernel to log connection tracking invalid message. So statement for [log1] and [emu1] is untouched.

```ini
# Logging of system packet through NFLOG
[log1]
# netlink multicast group (the same as the iptables --nflog-group param)
# Group O is used by the kernel to log connection tracking invalid message
group=0
#netlink_socket_buffer_size=217088
#netlink_socket_buffer_maxsize=1085440
# set number of packet to queue inside kernel
#netlink_qthreshold=1
# set the delay before flushing packet in the queue inside kernel (in 10ms)
#netlink_qtimeout=100

[emu1]
file="/var/log/ulog/syslogemu.log"
sync=1
```

并在配置文件的末尾为[log2]和[log3]添加另外两条语句。[emu2]和[emu3]也是如此

And add two another statement for [log2] and [log3] in the end of config file. The same is for [emu2] and [emu3]

```ini
[log2]
group=2
[emu2]
file="/var/log/ulog/group2-icmp.log"
sync=1

[log3]
group=3
[emu3]
file="/var/log/ulog/group3-ssh.log"
sync=1
```

这里是记录icmp协议的信息，我们在nftables里，需要添加log规则

```json
table inet my_icmp {
        chain input {
                type filter hook input priority 0; policy accept;
                ip protocol icmp log prefix "icmp-accept " group 2 accept
        }

        chain output {
                type filter hook output priority 0; policy accept;
                ip protocol icmp log prefix "icmp-accept " group 2 accept
        }
}

```


## 配置文件详解


`/etc/ulogd2.conf`

配置文件是个`ini`风格的文本，节（section）分成两类，一种是全局节`[global]`，整个配置文件只有一个，它定义了要加载的插件，一种是针对插件参数的实例化的节（一个插件可以实例化多次）

### [global] 全局节参考

`[global]`节主要的参数有

`logfile`、`loglevel`、`plugin`、`stack`

`logfile` 为ulogd日志输出设置，它可能是：文件名（字符串）、syslog、stdout

`loglevel` 为日志等级，可能取值为1-8， 1=debug information, 3=informational messages, 5=noticable exceptional conditions, 7=error conditions, 8=fatal errors, program abort.

`plugin` 参数为插件.so文件的路径（字符串），plugin可以有多个，导入多个插件，插件分成3种类型，数据输入（采集），数据转换（例如IP原始数据转人类便于阅读的字符串），日志输出插件（例如输出到sql数据，json文本，或者xml之类的），插件选项可以出现多次。



`stack` 参数可以定义多个日志栈，它包含多个插件的实例化对象，这个选项后面跟着一个插件实例列表，它将以输入插件开始，包含可选的过滤插件和输出插件结束。此选项可能出现多次。它大概长这样的

```ini
stack=log2:NFLOG,base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,mac2str1:HWHDR,json1:JSON
```

log2:NFLOG  log2是log2节的配置文件，NFLOG则是插件名称，意思是这个是通过`[log2]`的节实例化的NFLOG插件，stack以输入插件开始的

base1:BASE,ifi1:IFINDEX,ip2str1:IP2STR,mac2str1:HWHDR 这些则定义了一堆数据转换的插件

json1:JSON最后这个日志栈则以json插件配置的输出

### 插件节参考

首先插件节的名称并不重要，因为它只是在stack指定实例化插件的时候用到节的名称，这个名称可以根据自己需求更改

不同的插件，有基本的配置选项，这个选项可以通过运行ulogd2后，通过以下命令查看，不同的插件，可以根据/etc/ulogd2.conf里的plugin选项配置找到.so的具体位置，直接复制便可，一般ulogd2已经给出了默认的配置，只是它把这些配置注释掉了。下面是NFLOG的配置

```bash 
$ sudo ulogd -i /usr/lib/x86_64-linux-gnu/ulogd/ulogd_inppkt_NFLOG.so

Name: NFLOG
Config options:
        Var: bufsize (Integer, Default: 150000)
        Var: group (Integer, Default: 0)
        Var: unbind (Integer, Default: 1)
        Var: bind (Integer, Default: 0)
        Var: seq_local (Integer, Default: 0)
        Var: seq_global (Integer, Default: 0)
        Var: numeric_label (Integer, Default: 0)
        Var: netlink_socket_buffer_size (Integer, Default: 0)
        Var: netlink_socket_buffer_maxsize (Integer, Default: 0)
        Var: netlink_qthreshold (Integer, Default: 0)
        Var: netlink_qtimeout (Integer, Default: 0)
Input keys:
        Input plugin, No keys
Output keys:
        Key: raw.mac (raw data)
        Key: raw.pkt (raw data)
        Key: raw.pktlen (unsigned int 32)
        Key: raw.pktcount (unsigned int 32)
        Key: oob.prefix (string)
        Key: oob.time.sec (unsigned int 32)
        Key: oob.time.usec (unsigned int 32)
        Key: oob.mark (unsigned int 32)
        Key: oob.ifindex_in (unsigned int 32)
        Key: oob.ifindex_out (unsigned int 32)
        Key: oob.hook (unsigned int 8)
        Key: raw.mac_len (unsigned int 16)
        Key: oob.seq.local (unsigned int 32)
        Key: oob.seq.global (unsigned int 32)
        Key: oob.family (unsigned int 8)
        Key: oob.protocol (unsigned int 16)
        Key: oob.uid (unsigned int 32)
        Key: oob.gid (unsigned int 32)
        Key: raw.label (unsigned int 8)
        Key: raw.type (unsigned int 16)
        Key: raw.mac.saddr (raw data)
        Key: raw.mac.addrlen (unsigned int 16)
        Key: raw (raw data)

```

上面输出可以看到插件名称为`NFLOG`,这个名称我们在`[global]`节的stack参数需要用来实例化插件。

`Config options:`下的是该插件选项，以及默认值，选项的类型

`Input keys:`下的是这个插件接受的输入字段，输入插件没有输入字段。

`Output keys:`是这个插件输出的字段，这里的是一个输入插件，它输出的字段比较多。


我们可以通过这个信息构建一个插件的配置
```ini
[mynflog]
group = 2
```
输入插件具体的配置解释，需要参考文档，这里的group是log分组，也就是mynflog实例化后的插件，它只接受group为2的分组，而这个分组，在nftables里log规则指定group参数。

这样我们可以新建一个日志栈

```
stack=mynflog:NFLOG, xxxxxxxxxxx（紧接着是filter类型的插件实例）, 最后的是输出插件实例
```



---

## 参考文章：

[ulogd2文档](http://www2.kangran.su/~nnz/pub/nf-doc/ulogd2/)

[如何使用ulogd2记录nftables日志输出](https://www.mybluelinux.com/how-nftables-log-to-external-file/)
