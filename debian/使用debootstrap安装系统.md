# 使用debootstrap安装debian系统笔记

准备安装镜像：[debian-live-10.9.0-amd64-standard.iso 945M](https://mirrors.tuna.tsinghua.edu.cn/debian-cd/10.9.0-live/amd64/iso-hybrid/debian-live-10.9.0-amd64-standard.iso)

需要额外安装的软件包有

```c
linux-image-amd64 -> debian内核镜像包
systemd -> 新型的init程序
grub-efi-amd64 -> 安装efi启动的grub
makedev -> 创建/dev下的设备节点
locales -> 语言包
vim -> 文本编辑器
openssh-server -> ssh服务
sudo -> 临时授权
bash-completion -> bash Tab补全
```

前期先制作启动盘，对于支持UEFI启动的PC，直接把iso内容解压到一个fat32格式的U盘根目录即可（保证U盘根目录下有个EFI文件夹），或者使用制作工具把iso制作成u盘启动盘，对于已有的debian系统给别的硬盘安装debian，则单独安装 **bootstrap** 即可。以下以iso基础镜像安装为例的笔记。


# 前期准备

## 1、启动进入临时的debian系统

启动菜单后，进入standard镜像的一个live系统，它是一个没有GUI的临时维护系统，拥有debian的基础软件。启动后选择：

```
Debian GNU/Linux Live(kernel xxxxx)
```

启动系统后，进入系统，切换root用户：

```bash
sudo su
```



## 2、确认网络状态

debian的live镜像使用 **ifupdown**包的 **networking.service** 管理网络，在DHCP下的路由下自动分配IP比较容易管理。

启动系统后直接就可以通过DHCP获取到IP，如果特殊情况下没有网，或者需要手动管理。

如果需要手动管理网络，以下为使用 **systemd-networkd.service** 代替 **networking.service** 管理网络的配置。

先查看网卡和MAC地址：

```bash
root@debian:~# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:1f:1b:27 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::215:5dff:fe1f:1b27/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:1f:1b:29 brd ff:ff:ff:ff:ff:ff
    inet 192.168.33.182/24 brd 192.168.33.255 scope global dynamic eth1
       valid_lft 35935sec preferred_lft 35935sec
    inet6 fe80::215:5dff:fe1f:1b29/64 scope link
       valid_lft forever preferred_lft forever
```

此处找到有2个网卡（序号2和3）eht0、eth1，mac地址为 **00:15:5d:1f:1b:27** 、 **00:15:5d:1f:1b:29** 

新建一个systemd网络管理的模板 **/etc/systemd/network/01-lan.network** ，通过MAC地址匹配，这是DHCP分配IP的模板，一个网卡一个配置文件。

```ini
[Match]
MACAddress=00:15:5d:1f:1b:27

[Network]
DHCP=yes
```

如果是静态IP则需要指定IP（ **192.168.1.2** ）、网关( **192.168.1.1** )、DNS（ **223.5.5.5** ），

```ini
[Match]
MACAddress=00:15:5d:1f:1b:27

[Network]
Address=192.168.1.2
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
  2 eth0             ether              degraded    configuring
  3 eth1             ether              routable    configured
```

绿色的 **routable** 、说明链路正常，此处用eth1上网。多个网卡上网优先级，使用`Metric=` 设定跃点值，跃点值越低，优先级越高。

## 3、更换软件换源

使用 **nano** 编辑 **/etc/apt/sources.list** 

```bash
root@debian:~# nano  /etc/apt/sources.list
```

替换成清华源的

```sources.list
deb http://mirrors.tuna.tsinghua.edu.cn/debian/ buster main contrib non-free
```

 **ctrl+o** 保存文件，**ctrl+x**  退出nano，更新索引

```bash
root@debian:~# apt update
```

## 4、安装装系统时需要的软件

debootstrap

```bash
root@debian:~# apt install debootstrap dosfstools
```

> dosfstools 用于格式化vfat格式分区（EFI分区）
>
> debootstrap 用于安装新系统



## 5、使用fdisk对硬盘进行分区与格式化

 **格式化硬盘前应该备份数据**

确定硬盘

```bash
root@debian:/mnt# lsblk
NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0   7:0    0 611.2M  1 loop /usr/lib/live/mount/rootfs/filesystem.squashfs
sda     8:0    0    50G  0 disk
sr0    11:0    1   945M  0 rom  /usr/lib/live/mount/medium
```

发现sda硬盘就是需要装系统的硬盘

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

Created a new partition 1 of type 'Linux filesystem' and of size 300 MiB.
Partition #1 contains a vfat signature.

Do you want to remove the signature? [Y]es/[N]o: Y

The signature will be removed by a write command.

Command (m for help): t
Selected partition 1
Partition type (type L to list all types): 1
Changed type of partition 'Linux filesystem' to 'EFI System'.
```

- n命令创建主分区
- 选择分区编号，默认为1，直接回车
- 选择分区起始扇区，默认为2048，直接回车
- 选择分区的结束扇区，用+/-可以通过大小来设定，+300M，创建一个300M的分区
- 由于此前已经创建了一个EFI分区，移除signature
- t命令更改分区标识，输入1，回车，更改成EFI类型

剩余的空间创建linux根目录的分区

```bash
Command (m for help): n
Partition number (2-128, default 2):
First sector (616448-104857566, default 616448):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (616448-104857566, default 104857566):

Created a new partition 2 of type 'Linux filesystem' and of size 49.7 GiB.
Partition #2 contains a ext4 signature.

Do you want to remove the signature? [Y]es/[N]o: Y

The signature will be removed by a write command.

Command (m for help):
```



- n命令创建主分区，回车
- 选择分区编号，第二个分区默认为2，直接回车
- 选择分区的起始扇区和结束扇区，直接回车即可
- 由于此前已经创建了一个EFI分区，移除signature

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

在/mnt下创建挂载点，root文件夹为根目录的挂载点

```bash
root@debian:/mnt# mkdir /mnt/root
```

挂载

```bash
root@debian:/mnt# mount -t ext4 -o defaults,rw /dev/sda2 /mnt/root/
```



## 7、使用debootstrap下载基本系统并释放文件

使用debootstrap工具构建一个基础的debian系统。镜像选择清华源。官网参考用法

```
debootstrap 版本名 ./释放目录 镜像网址
debootstrap wheezy ./wheezy-chroot http://http.debian.net/debian/
```

以buster为例

```bash
debootstrap --arch=amd64 buster /mnt/root https://mirrors.tuna.tsinghua.edu.cn/debian/
```

因为需要安装一些软件包，所以包含进去，最终这样的

```bash
debootstrap --arch=amd64 --include=linux-image-amd64,systemd,grub-efi-amd64,makedev,locales,vim,openssh-server,man,sudo,bash-completion buster /mnt/root https://mirrors.tuna.tsinghua.edu.cn/debian/
```

如果速度太慢，也可以用网易云的,如下：

```bash
debootstrap --arch=amd64 --include=linux-image-amd64,systemd,grub-efi-amd64,makedev,locales,vim,openssh-server,sudo,bash-completion buster /mnt/root http://mirrors.163.com/debian/
```

 **debootstrap** 执行成功后提示，这个下载过程可能很漫长。

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
root@debian:/mnt/root# mount -t proc proc /mnt/root/proc
root@debian:/mnt/root# mount -t sysfs sysfs /mnt/root/sys
```

设置locales

```bash
root@debian:~#  dpkg-reconfigure locales
```

选择语言为 **en_US.UTF-8** ，切换根目录，更新软件索引

```bash
root@debian:/mnt/root# chroot /mnt/root/
root@debian:~# apt updateapt update
```

创建/dev下的设备，在安装grub的时候有用到。

```bash
root@debian:/# cd /dev
root@debian:/dev# MAKEDEV sda
```



## 10、安装BootLoader

将grub安装到EFI分区

```bash
root@debian:~# grub-install -v  --efi-directory=/boot/EFI  --boot-directory=/boot --no-uefi-secure-boot --target=x86_64-efi
```

安装成功后提示

```
Installation finished. No error reported.
```

编辑grub配置文件。

```
nano vim /etc/default/grub
```

修改grub参数，注意root分区的UUID和新分区的uuid相同

```
GRUB_CMDLINE_LINUX_DEFAULT="vga root=UUID=\"1a909d07-88bb-4419-8e53-9640287111dd\" init=/usr/bin/systemd ro vga vt"
```

更新grub配置，更新initramfs

```
root@debian:~# update-initramfs -u
root@debian:~# update-grub
```



## 11、对新系统进行设置

临时更改root用户密码

```bash
root@debian:~# passwd root
```

新系统因为没有配置网络，是没有网卡的配置，此处换回systemd-networkd

配置新系统的网络接口（使用 `systemd-networkd` + `networkctl`  管理）：

```ini
# /etc/systemd/network/01-lan.network
[Match]
MACAddress=00:15:5d:1f:1b:27

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



## 12、卸载文件系统、关闭临时系统

```bash
# chroot退出前,磁盘缓存回写，卸载proc、sys、EFI分区
root@debian:~# sync
root@debian:~# sync
root@debian:~# sync
root@debian:~# cd /
root@debian:/# umount /proc
root@debian:/# umount /sys
root@debian:/# umount /boot/efi
root@debian:/# exit
# 退出后
root@debian:/mnt/root# cd /
root@debian:/# umount /mnt/root
root@debian:/# systemctl poweroff
```



## 13、启动新系统及配置

启动后登陆系统root用户。

由于新系统中没有使用 **ifupdown** 的 **networking.service**  管理网络，使用 **systemd-networkd** ,参考临时系统配置网络的方法，**/etc/systemd/network/01-lan.network** 内容：

```ini
[Match]
MACAddress=00:15:5d:1f:1b:27

[Network]
DHCP=yes
```

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
       Address: 192.168.33.183 on eth0
                fe80::215:5dff:fe1f:1b2a on eth0
       Gateway: 192.168.33.1 (Shenzhen Winyao Electronic Limited) on eth0
           DNS: 192.168.33.1
           NTP: 203.107.6.88
```

可以尝试ping一个网址检测网络连通

```bash
root@debian:~# ping www.baidu.com
PING www.a.shifen.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=1 ttl=55 time=8.68 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=2 ttl=55 time=9.53 ms
```

到此，不带桌面环境的debian安装完毕，一般通过ssh登录操作。

# 14、安装桌面环境

安装桌面环境和显示管理器完成整个桌面安装，包比较多。这里安装cinnamon桌面与lightdm显示管理服务。

```bash
root@debian:~# apt install cinnamon lightdm
```

启动显示管理器后通过登录以启动桌面

```bash
root@debian:~# systemctl start lightdm
```



---



# 资料参考：

[deboostrap wiki](https://wiki.debian.org/zh_CN/Debootstrap)

[通过Unix/Linux系统安装debian wiki](https://www.debian.org/releases/stable/amd64/apds03.zh-cn.html)

[systemd 中文手册 (译者金步国)](http://www.jinbuguo.com/systemd/systemd.index.html)

[systemd.network 中文手册](http://www.jinbuguo.com/systemd/systemd.network.html#)

[systemctl 中文手册](http://www.jinbuguo.com/systemd/systemctl.html#)

[networkctl 中文手册](http://www.jinbuguo.com/systemd/networkctl.html#)

