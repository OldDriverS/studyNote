# 使用debootstrap安装debian系统笔记


## 什么是LiveCD?

Live 安装映像包含一个 Debian 系统，可以在不修改硬盘驱动器上的任何文件的情况下进行启动，并可根据映像内容安装 Debian。

所谓的 live 镜像，或是更精确地称为 live system （实况系统），指的是为 DVD、USB 闪存盘等媒介准备的镜像，含有已预先安装的完整系统。你不需要安装任何东西到硬盘上，相反地你可以直接从媒介（DVD 或 USB 闪存盘）上开机 而且可马上开始工作。所有的程序都直接从媒介上执行。

以上摘录debian的FAQ。

> 当前stable是bullseye,涉及软件包名称无变化，笔记仍然可用。

准备安装镜像：[debian-live-10.9.0-amd64-standard.iso 945M](https://opentuna.cn/debian-cd/10.9.0-live/amd64/iso-hybrid/debian-live-10.9.0-amd64-standard.iso)

standard 默认无桌面环境，如需要桌面可选带桌面版本的Live

此次方式涉及的其他软件包有：

```
linux-image-amd64  debian内核镜像包
systemd  新型的init程序
grub-efi-amd64  安装efi启动的grub
makedev  创建/dev下的设备节点脚本
locales  语言
vim  文本编辑器
openssh-server  ssh服务
sudo  临时授权
bash-completion  bash 补全提示
ca-certificates CA证书，https用到
```

制作USB启动介质：

通过刻录软件将ISO文件写到U盘介质。

对于支持UEFI启动的平台，不借助刻录软件，先将U盘文件系统格式化成FAT32格式，然后将ISOiso内容解压到一个fat32格式的U盘根目录即可（保证U盘根目录下有个EFI文件夹）


# 安装思路

- 下载一个debian livecd的iso镜像，此livecd包含debian基础的debian环境

- livecd刻录到U盘，启动U盘中的debian livecd系统

- livecd系统配网

- 更换livecd中apt源为国内源

- 从源中获取安装debootstrap及系统安装需要的相关软件

- 在硬盘中，为新的debian系统分区并创建文件系统

- 挂载文件系统，并使用deboostrap释放文件

- 通过chroot临时切换到新系统的环境

- 配置新系统

- 安装新系统UEFI启动的grub引导

- 保存，卸载分区

- 重启进新系统

- 安装完毕


# 前期准备

## 1、启动进入临时的debian系统

启动菜单后，进入standard镜像的一个live系统，纯命令行操作，拥有debian的基础软件。启动后选择：

```
Debian GNU/Linux Live(kernel xxxxx)
```

启动系统后，切换root用户：

```bash
sudo su
```



## 2、确认网络状态

共有三种网络管理软件可供选择

- systemd-networkd  属于systemd的一部分
- ifupdown   传统的网管软件
- NetworkManager  桌面端常用的网管软件

### 2.1 使用 systemd-networkd 配置网络

以下为使用 **systemd-networkd.service** 代替 **networking.service** 管理网络的配置。

先查看网卡和MAC地址：

```bash
root@debian:~# ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
    link/ether 52:54:00:59:b4:a0 brd ff:ff:ff:ff:ff:ff
```

有2个网卡（序号1和2）：lo、enp1s0

mac地址为 **52:54:00:59:b4:a0** 

新建一个systemd网络管理的模板 **/etc/systemd/network/01-lan.network** ，该配置文件仅通过MAC地址匹配接口。

情况1：使用dhcp分配IP

```ini
[Match]
MACAddress=52:54:00:59:b4:a0

[Network]
DHCP=yes
```
情况2：使用静态IP设置

如果是静态IP则需要指定IP（ **192.168.1.2** ）、网关( **192.168.1.1** )、DNS（ **223.5.5.5** ），

```ini
[Match]
MACAddress=52:54:00:59:b4:a0

[Network]
Address=192.168.1.2/24
Gateway=192.168.1.1
DNS=223.5.5.5
```

保存后，启动systemd-networkd服务

```bash
systemctl start systemd-networkd
```

查看网络管理服务状态

```bash
root@debian:~# systemctl status systemd-networkd
● systemd-networkd.service - Network Service
   Loaded: loaded (/lib/systemd/system/systemd-networkd.service; disabled; vendor preset: enabled)
   Active: active (running) since Thu 2021-05-20 14:10:53 UTC; 9min ago
     Docs: man:systemd-networkd.service(8)
 Main PID: 2928 (systemd-network)
   Status: "Processing requests..."
    Tasks: 1 (limit: 4681)
   Memory: 4.3M
   CGroup: /system.slice/systemd-networkd.service
           └─2928 /lib/systemd/systemd-networkd
```

 **systemd-networkd** 状态正常，通过 **networkctl** 查看网络状态

```bash
root@debian:~# networkctl
IDX LINK             TYPE               OPERATIONAL SETUP
  1 lo               loopback           carrier     unmanaged
  2 enp1s0              ether           routable    configured
```

**routable** 、说明链路正常，此处用enp1s0上网。

### 2.2 使用networking(ifupdown)配置网络

live镜像使用 **ifupdown** 包的 **networking.service** 服务管理网络，在DHCP下的路由下自动分配IP比较容易管理。

live系统启动后先通过DHCP获取到IP，如果特殊情况下没有网，可能需要手动管理。

修改 `/etc/network/interfaces`

情况1：使用dhcp分配

```
auto enp1s0
iface enp1s0 inet dhcp
```

情况2：静态指定IP

```
auto enp1s0
iface enp1s0 inet static
    address 192.168.1.2/24
    gateway 192.168.1.1
```

- auto 让指定接口在networking启动时配置生效
- iface 指定该接口的参数


修改DNS服务器地址

```
# /etc/resolv.conf
nameserver 223.5.5.5
```


## 3、更换软件换源

使用 **nano** 编辑 **/etc/apt/sources.list** 

```bash
root@debian:~# nano  /etc/apt/sources.list
```

替换成opentuna的， **ctrl+o** 保存文件，**ctrl+x**  退出nano

```sources.list
deb http://opentuna/debian/ buster main contrib non-free
```

更新索引

```bash
root@debian:~# apt update
```

## 4、安装系统时需要的软件


```bash
root@debian:~# apt install debootstrap dosfstools
```

> dosfstools 用于格式化vfat格式（用于EFI分区）
>
> debootstrap 用于安装新系统



## 5、使用fdisk对硬盘进行分区与格式化

 **注意：格式化硬盘前应该备份数据！！！**

### 5.1 确定安装系统的硬盘

> Tips：sda是sata接口第一块硬盘，如果安装系统在nvme,则是nvme0n1,如果是虚拟机，半虚拟化硬盘virtio驱动则是vda,不同接口的块设备命名是不一样的，根据实际情况调整。

```bash
root@debian:/mnt# lsblk
NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0   7:0    0 611.2M  1 loop /usr/lib/live/mount/rootfs/filesystem.squashfs
sda     8:0    0    50G  0 disk
sr0    11:0    1   945M  0 rom  /usr/lib/live/mount/medium
```

根据容量可以区分，发现sda硬盘就是需要装系统的硬盘

> 如果容量没法区分，可以根据SMART信息确认硬盘

```
# apt install smartmontools
# # smartctl -i /dev/sdc
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.10.0-18-amd64] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Model Family:     Samsung based SSDs
Device Model:     Samsung SSD 860 EVO 250GB
Serial Number:    S4CK********63V
LU WWN Device Id: 5 002538 e7022bfeb
Firmware Version: RVT03B6Q
User Capacity:    250,059,350,016 bytes [250 GB]
Sector Size:      512 bytes logical/physical
Rotation Rate:    Solid State Device
Form Factor:      2.5 inches
TRIM Command:     Available, deterministic, zeroed
Device is:        In smartctl database [for details use: -P show]
ATA Version is:   ACS-4 T13/BSR INCITS 529 revision 5
SATA Version is:  SATA 3.2, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Sat Oct 29 11:15:18 2022 CST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled
```
例如以上是一块三星860 EVO 250GB，可以根据硬盘的型号，SN，容量，厂商等，情况区分不同的硬盘，选择一个目的硬盘。


```
root@debian:/mnt# sudo fdisk /dev/sda

Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help):
```

进入fdisk命令行界面。按m键回车获得命令行提示，先创建GPT分区表

```
Command (m for help): g
Created a new GPT disklabel (GUID: D53ADE4F-6160-824F-B517-15DC2D513E3E).
```

创建一个300M的主分区

```bash
Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-104857566, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-104857566, default 104857566): +300M

Command (m for help): t
Selected partition 1
Partition type (type L to list all types): 1
Changed type of partition 'Linux filesystem' to 'EFI System'.
```

- n命令创建主分区
- 选择分区编号，默认为1，直接回车
- 选择分区起始扇区，默认为2048，直接回车
- 选择分区的结束扇区，用+/-可以通过大小来设定，+300M，创建一个300M的分区
- t命令更改分区标识，输入1，回车，更改成EFI类型

剩余的空间创建linux根目录的分区

```bash
Command (m for help): n
Partition number (2-128, default 2):
First sector (616448-104857566, default 616448):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (616448-104857566, default 104857566):

Command (m for help):
```

- n命令创建主分区，回车
- 选择分区编号，第二个分区默认为2，直接回车
- 选择分区的起始扇区和结束扇区，直接回车即可

```bash
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

w命令真正写入分区表后退出fdisk，查看已有的硬盘，针对硬盘/dev/sda分区，

```bash
root@debian:/mnt# fdisk -l /dev/sda
Disk /dev/sda: 50 GiB, 53687091200 bytes, 104857600 sectors
Disk model: Virtual Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: D53ADE4F-6160-824F-B517-15DC2D513E3E

Device      Start       End   Sectors  Size Type
/dev/sda1    2048    616447    614400  300M EFI System
/dev/sda2  616448 104857566 104241119 49.7G Linux filesystem
```

格式化分区

```bash
root@debian:/mnt# mkfs.vfat /dev/sda1
root@debian:/mnt# mkfs.ext4 /dev/sda2
```

## 6、挂载分区

创建`/mnt/root` 目录作为新系统根分区的临时挂载点，并挂载。

```bash
root@debian:/mnt# mkdir /mnt/root
root@debian:/mnt# mount -t ext4 -o defaults,rw /dev/sda2 /mnt/root/
```


## 7、使用debootstrap下载基本系统并释放文件

```
# debootstrap --help

Usage: debootstrap [OPTION]... <suite> <target> [<mirror> [<script>]]
Bootstrap a Debian base system into a target directory.

```

- suite 版本代号，例如 buster
- target 释放的目的文件夹 例如 /mnt/root
- mirror 景象站的地址，例如 http://opentuna.cn/debian

使用debootstrap工具构建一个基础的debian系统。镜像选择opentuna。官网有参考用法

```
debootstrap 版本名 ./释放目录 镜像网址
debootstrap wheezy ./wheezy-chroot http://http.debian.net/debian/
```

以debian 10 buster为例，如果是其他版本，则需要将 `buster` 替换成对应的版本代码，例如，debian 11 是bullseye

```bash
debootstrap --arch=amd64 buster /mnt/root https://opentuna.cn/debian/
```

新系统需要额外包含一些软件包，所以最终命令是这样的

```bash
debootstrap --arch=amd64 --include=linux-image-amd64,systemd,grub-efi-amd64,makedev,locales,vim,openssh-server,sudo,bash-completion,ca-certificates buster /mnt/root https://opentuna.cn/debian/
```

**debootstrap** 执行成功后提示，这个下载过程可能很漫长，具体看宽带和硬盘。

````
I: Base system installed successfully.
````

创建 **EFI** 挂载点，并挂载

```bash
root@debian:/mnt/root# mkdir /mnt/root/boot/EFI
root@debian:/mnt/root# mount -t vfat -o defaults,rw /dev/sda1 /mnt/root/boot/EFI/
```

## 8、编辑文件系统挂载配置

查看分区的ID

```bash
root@debian:/mnt/root# blkid
/dev/sr0: UUID="2021-03-27-14-49-37-00" LABEL="d-live 10.9.0 st amd64" TYPE="iso9660" PTUUID="30d6eb01" PTTYPE="dos"
/dev/loop0: TYPE="squashfs"
/dev/sda1: SEC_TYPE="msdos" UUID="7D8E-ABEB" TYPE="vfat" PARTUUID="52089f8b-ee5d-5e4d-ae74-3249d10cc478"
/dev/sda2: UUID="1a909d07-88bb-4419-8e53-9640287111dd" TYPE="ext4" PARTUUID="a715432e-436a-6e45-ac60-eab59f2dff26"
```

创建fstab，注意替换UUID

```bash
root@debian:/mnt/root# vim /mnt/root/etc/fstab
```

 **/etc/fstab** 内容如下：**#** 开头的为注释，为后续添加挂载点方便，还是添加一个格式注释。

```fstab
# UNCONFIGURED FSTAB FOR BASE SYSTEM
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
UUID=1a909d07-88bb-4419-8e53-9640287111dd /               ext4    errors=remount-ro 0       1
UUID=7D8E-ABEB  /boot/efi       vfat    umask=0077      0       1
```


## 9、chroot到新系统根目录

chroot前先挂载特殊的文件系统/proc /dev /sys等

```bash
mount -t proc proc /mnt/root/proc
mount -t sysfs sysfs /mnt/root/sys
mount -o bind /dev /mnt/root/dev
```

一般情况下，可以直接把/dev目录挂载到/mnt/root/dev，这样最省事

安装grub的时候有用到。

也可以用mknod一个个设备节点创建出来，或者用MAKEDEV批量创建。

```bash
root@debian:/# cd /dev
root@debian:/dev# MAKEDEV sda
```

还是用 `mount -o bind /dev /mnt/root/dev` 最省事，就不需要使用MAKEDEV了


```bash
root@debian:/mnt/root# chroot /mnt/root/
root@debian:~# apt updateapt update
```

设置locales，这是语言相关的包

```bash
root@debian:~#  dpkg-reconfigure locales
```

选择语言为 **en_US.UTF-8** ，因为tty对中文的支持不好，可以后续再切换成 **zh_CN.utf-8**


## 10、安装引导

使用grub引导系统启动，grub会先载入内核和initrd

initrd在内存展开一个临时的根分区，里面有一些早期初始化的程序和部分内核模块。

然后在进一步挂载硬盘上的根分区。

将grub安装到EFI分区

```bash
root@debian:~# grub-install -v  --efi-directory=/boot/EFI  --boot-directory=/boot --no-uefi-secure-boot --target=x86_64-efi
```

安装过程中，log较多，安装成功后提示

```
Installation finished. No error reported.
```

编辑grub配置文件，该文件会影响update-grub生成/boot/grub/grub.cfg。

```
nano /etc/default/grub
```

修改内核命令行参数，注意root分区的UUID和新系统根分区的uuid相同，指定使用的init,这里使用systemd

```
GRUB_CMDLINE_LINUX_DEFAULT='root=UUID="1a909d07-88bb-4419-8e53-9640287111dd" init=/usr/bin/systemd ro vga vt'
```

更新grub配置，更新initramfs

```
root@debian:~# update-initramfs -u
root@debian:~# update-grub
```

emmm...只是该配置不需要update-initramfs,如果根分区使用了软raid,在这个引导阶段，需要把mdadm相关的脚本包含进去，让RAID完成装载，才能进一步挂载RAID上的根分区。


这个过程当然不用自己写，debian已经包含了这部分内容，只需要更新进去就好啦。

## 11、对新系统进行设置

更改新系统root用户密码

```bash
root@debian:~# passwd root
```

新系统因为没有配置网络，没有网卡的配置，此处换回systemd-networkd

配置新系统的网络接口（使用 `systemd-networkd` + `networkctl`  管理）：

```ini
# /etc/systemd/network/01-lan.network
[Match]
MACAddress=52:54:00:59:b4:a0

[Network]
DHCP=yes
```

修改允许root登陆ssh

```
root@debian:/etc/ssh# vim /etc/ssh/sshd_config
```

添加允许root用户登录的参数，方便后续新系统启动后进一步设置。

```ini
PermitRootLogin yes
```

这个只是方便启动系统后，远程ssh进系统进行后续的配置。。该完后应该把这个配置删掉，ssh不应该让root用户登陆，而是指定一个普通用户，使用sudo提权。


## 12、卸载文件系统、关闭临时系统

```bash
## chroot退出前,磁盘缓存回写，卸载proc、sys、EFI分区
root@debian:~# sync
root@debian:~# sync
root@debian:~# sync
root@debian:~# cd /
root@debian:/# umount /proc
root@debian:/# umount /sys
root@debian:/# umount /boot/efi
root@debian:/# umount /dev
root@debian:/# exit
## 退出chroot后 ##
root@debian:/mnt/root# cd /
root@debian:/# umount /mnt/root
root@debian:/# systemctl poweroff
```


# 启动新系统及配置

## 1、配置网络

启动后登陆系统root用户。

由于新系统中没有使用 **ifupdown** 的 **networking.service**  管理网络，使用 **systemd-networkd** ,参考临时系统配置网络的方法，**/etc/systemd/network/01-lan.network** 内容：

```ini
[Match]
MACAddress=52:54:00:59:b4:a0

[Network]
DHCP=yes
```

> 如果需要配置静态IP，参考前面在liveCD里配置网络的方法。


处理系统服务，允许以下服务启动，并启动网络服务：

- systemd-networkd
- ssh
- systemd-resolvd

```bash
root@debian:~# systemctl enable systemd-networkd ssh systemd-resolvd
root@debian:~# systemctl start systemd-networkd
```

通过networkctl确认网络状态。

```bash
root@debian:~# networkctl status
●        State: routable
        Address: 192.168.1.2 on enp1s0
        Gateway: 192.168.1.1 on enp1s0

```

可以尝试ping一个网址检测网络连通

```bash
root@debian:~# ping www.baidu.com
PING www.a.shifen.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=1 ttl=55 time=8.68 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=2 ttl=55 time=9.53 ms
```

到此，不带桌面环境的debian安装完毕，一般通过ssh登录操作。

## 装桌面环境

安装桌面环境和显示管理器完成整个桌面安装，包比较多。这里安装kde桌面与sddm显示管理服务。

```bash
root@debian:~# apt install kde-standard sddm
```

启动显示管理器后通过登录以启动桌面

```bash
root@debian:~# systemctl start sddm
```

也可以通过systemd对dm服务的软连接启动（例如安装sddm后，display-manager.service默认是指向sddm）

```bash
root@debian:~# systemctl start display-manager
```

kde还有别的组件（其他桌面环境在打包时，相关组件也可能分好几个包），通过以下 `apt search` 或者 `aptitude search` 命令搜索，根据需要安装

```bash
root@debian:~# apt install aptitude
root@debian:~# aptitude search ^kde-
i A kde-baseapps                                    - base applications from the official KDE release (metapack
i A kde-cli-tools                                   - tools to use KDE services from the command line
i A kde-cli-tools-data                              - tools to use kioslaves from the command line
p   kde-config-cddb                                 - CDDB retrieval configuration
p   kde-config-cron                                 - program scheduler frontend
p   kde-config-fcitx                                - 小企鹅输入法的 KDE 配置模块
i   kde-config-fcitx5                               - KDE configuration module for Fcitx5
......后面省略

```

---

# 使用 **network-manager** 作为网络管理

目前在Debian下可以选择的网络管理服务有：

- systemd-networkd ： 属于systemd的一部分，但是这个貌似没有一个友好的GUI管理客户端，总靠配置文件来设置网络
- ifupdown :  也就是教程中出现频率最高的networking服务，它包含了/etc/network/interfaces之类的配置
- network-manager : 这种在桌面用户用的应该比较多。

如果使用桌面，可以考虑network-manager作为网络管理软件，它包含1个服务端或多种类型的客户端（CUI、TUI、GUI）

- NetorkManager.service 服务

- 多种类型的客户端

- 涵盖多种网络、以太网、pppoe拨号、wifi等等。

- 有图形化的配置或者终端的图形客户端，配置起来非常灵活

    

> 不同的网络管理软件可以共存，但对于同一个网络接口，网络管理软件应该只使用一个，避免多次配置，相互竞争产生混乱，当systemd-networkd接口接管了某个接口，NetworkManager可以设置成unmanaged

无论是systemd-networkd或者是network-manager他们的客户端与服务端通讯，都将通过dbus。像systemd这玩意是极度依赖dbus的。



## 安装network-manager

可以在两个时间节点上安装：

在debootstrap释放准系统的时候。

```bash
debootstrap --arch=amd64 --include=linux-image-amd64,systemd,grub-efi-amd64,makedev,locales,vim,openssh-server,sudo,bash-completion buster /mnt/root https://opentuna.cn/debian/
```

如果已经错过了，已经释放了基础系统，可以chroot后通过apt安装network-manager，然后禁用另外两种网络管理服务

```bash
apt install network-manager
apt purge ifupdown
systemctl disable systemd-networkd networking
systemctl stop systemd-networkd networking
systemctl enable NetworkManager
systemctl start NetworkManager
```

## NetworkManager 使用

它有以下几个客户端

```
# GUI
# nm-applet是配合gnome一起使用的，并不能直接执行启动图形。他们属于network-manager-gnome包
nm-applet
nm-conection-editor

# CUI
nmcli
nm-online

# TUI
nmtui
# nmtui包含了以下组件
nmtui-connect
nmtui-edit
nmtui-hostname
```

根据情况可以选择不同的客户端来设置网络服务。

*nmcli* *nmtui* 均可在命令行下使用。

*nm-applet* 则在 桌面环境使用

*nm-online* 用于检测网络在线，可以检测当前网络是否连通或者当前NetworkManager服务是否启动。

注意，如果NetworkManager服务未启动，无法使用以上命令，或者图形的客户端提示网络离线。


# 将系统安装到RAID1上

至少需要2块及以上的硬盘才能组RAID1，其他安装流程是类似的，主要的区别是在储存结构和系统引导上。

## 1.储存结构

两块硬盘 vda vdb
```
NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
vda       254:0    0   20G  0 disk  
├─vda1    254:1    0  300M  0 part  
└─vda2    254:2    0 19.7G  0 part  
  └─md127   9:127  0 19.7G  0 raid1 /
vdb       254:16   0   20G  0 disk  
├─vdb1    254:17   0  300M  0 part  
└─vdb2    254:18   0 19.7G  0 part  
  └─md127   9:127  0 19.7G  0 raid1 /
```

每块硬盘上都有2个分区，vda1 vdb1各分300M作为EFI分区，vda2 vdb2 组成软RAID1，用作根分区。

在debootstrap释放系统前，需先创建RAID1阵列，使用mdadm管理软raid

```
mdadm [mode] <raiddevice> [options] <component-devices>
```

- mode 该命令分成好几个模式，例如创建阵列，
- raiddevice raid设备节点，例如/dev/md0
- options 不同模式的参数也不一样
- component-devices 涉及的块设备，例如/dev/vda2


例如创建RAID需要制定的参数有
- raid-devices 组成RAID有几块硬盘，例如2
- level RAID阵列的级别，例如raid1
- name 阵列的别称，他会在/dev/md/name创建对应的节点
- homehost 阵列的主机标识，用any可以忽略

```
mdadm --create /dev/md0 --raid-devices 2 --level raid1 --name root --homehost any  /dev/vda2 /dev/vdb2
```

执行后，查看raid1同步情况，他会复制数据以保证两块硬盘数据相同（这里其实是分区）

```
cat /proc/mdstat 
Personalities : [raid1] [linear] [multipath] [raid0] [raid6] [raid5] [raid4] [raid10] 
md0 : active raid1 vdb2[1] vda2[0]
      31145984 blocks super 1.2 [2/2] [UU]
      [>....................]  resync =  2.3% (742720/31145984) finish=2.7min speed=185680K/sec
```

同步的速度取决于硬盘的读写速度和RAID的大小

未同步完成已经可以操作阵列了，格式化文件系统

```
mkfs -t ext4 /dev/md0
```

挂载根分区

```
mkdir /mnt/root
mount -t ext4 -o rw /dev/md0 /mnt/root
```

debootstrap安装过程和非阵列的情况一样，释放基础系统，配置网络，用户。

```
debootstrap --arch=amd64 --include=linux-image-amd64,systemd,grub-efi-amd64,makedev,locales,vim,openssh-server,sudo,bash-completion,ca-certificates,mdadm bullseye /mnt/root https://opentuna.cn/debian/
```
软件包需要包含mdadm，因为mdadm包中有initramfs加载mdraid的脚本，需要打包进去的。

在格式化EFI分区、安装grub的时候有点区别,因为有两个EFI分区

都格式化了

```
mkfs -t vfat /dev/vda1
mkfs -t vfat /dev/vdb1
```
主要使用vda的EFI分区做引导，vdb1备用，一个个安装grub

```
mkdir /mnt/root/boot/efi
mount -t vfat /dev/vda1 /mnt/root/boot/efi/
```

然后chroot切换到新的根分区

```
chroot /mnt/root
```

安装grub

```bash
grub-install -v --modules="part_gpt part_msdos lvm mdraid09 mdraid1x" --boot-directory=/boot  --efi-directory=/boot/efi --no-uefi-secure-boot  --target=x86_64-efi
```

因为grub.cfg是在RAID上的，所以编译出来的grubx64.efi需要包含部分模块，他才能载入RAID,并且从RAID里读取其他的grub模块和配置文件，添加到efi文件里的模块用--modules指定。

grub要指定给内核启动参数指定root分区的，因为md设备名称可能会变，还是用uuid区分

查看md0的UUID

```
root@debian:/boot# blkid /dev/md0 
/dev/md0: UUID="fcb3b088-d87b-47f9-ab65-dc114683e043" BLOCK_SIZE="4096" TYPE="ext4"
```

编辑 `/etc/fstab`，改变系统启动后挂载的根分区

```
UUID=fcb3b088-d87b-47f9-ab65-dc114683e043  /  ext4    errors=remount-ro 0       1
```

编辑 `/etc/default/grub` 修改内核启动参数

```
GRUB_CMDLINE_LINUX_DEFAULT='root=UUID="fcb3b088-d87b-47f9-ab65-dc114683e043" init=/usr/bin/systemd ro vga vt'
```

更新配置

```
update-grub
```

因为使用了RAID1，在initramfs阶段，需要使用到一些内核模块，让它在这个阶段配合mdadm脚本能装载md设备。

修改 `/etc/initramfs-tools/modules`，把raid1添加进去，去掉注释 s`#`

```
# List of modules that you want to include in your initramfs.
# They will be loaded at boot time in the order below.
#
# Syntax:  module_name [args ...]
#
# You must run update-initramfs(8) to effect this change.
#
# Examples:
#
raid1
# sd_mod

```

更新配置

```
update-initramfs -u
```


两块盘上都部署EFI分区，并且都装上grub,当启动过程中，任意一个盘掉盘了，另外一个盘上还是允许引导启动系统。

如果在运行过程中掉盘了，系统仍然不会崩溃，并可在线添加硬盘修复RAID阵列降级问题。

如果EFI没有识别启动条目，因为debian安装的引导的路径是`EFI/debian/grubx64.efi`，有些BIOS对UEFI的支持一般，他不会自动寻找其他文件夹的efi文件，导致无法识别启动条目。

- 按规范复制到 `EFI/BOOT/bootx64.efi`
- 通过efibootmgr,编辑nvram的启动条目，RAID两个硬盘的grub启动条目都新增进去

```
efibootmgr -v -c -L debian_main -l /EFI/debian/grub.efi -d /dev/vda
efibootmgr -v -c -L debian_backup -l /EFI/debian/grub.efi -d /dev/vdb
```

对他们启动顺序进行排序

```
root@debian:~#  efibootmgr  -o 4,6,2,1
BootCurrent: 0004
Timeout: 0 seconds
BootOrder: 0004,0006,0002,0001
Boot0000* UiApp
Boot0001* UEFI Misc Device
Boot0002* EFI Internal Shell
Boot0004* debian_main  
Boot0006* debian_backup
```



# 资料参考：

[deboostrap wiki](https://wiki.debian.org/zh_CN/Debootstrap)

[通过Unix/Linux系统安装debian wiki](https://www.debian.org/releases/stable/amd64/apds03.zh-cn.html)

[systemd 中文手册 (译者金步国)](http://www.jinbuguo.com/systemd/systemd.index.html)

[systemd.network 中文手册](http://www.jinbuguo.com/systemd/systemd.network.html#)

[systemctl 中文手册](http://www.jinbuguo.com/systemd/systemctl.html#)

[networkctl 中文手册](http://www.jinbuguo.com/systemd/networkctl.html#)

[mdadm 帮助手册](https://manpages.debian.org/bullseye/mdadm/mdadm.8.en.html)

[linux RAID wiki](https://raid.wiki.kernel.org/index.php/Linux_Raid)