# debian系统RAID及恢复

储存结构

```
# lsblk
NAME    FSTYPE        FSVER         LABEL                 UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda
├─vda1  vfat          FAT32                               37CE-3948
└─vda2  linux_raid_me 1.2           deraid:0              721fbb5a-b1f4-f09e-af34-e111dea5462c
  └─md0 ext4          1.0                                 b4d1ae73-d2ed-4cc7-a8e1-e13587c98dcc   26.4G     4% /
vdb
├─vdb1  vfat          FAT32                               37CE-CA34
└─vdb2  linux_raid_me 1.2           deraid:0              721fbb5a-b1f4-f09e-af34-e111dea5462c
  └─md0 ext4          1.0                                 b4d1ae73-d2ed-4cc7-a8e1-e13587c98dcc   26.4G     4% /
```

EFI启动，两个硬盘都单独存在一个EFI分区，将EFI文件夹复制到另外一个硬盘的EFI分区，当任意一个硬盘掉盘都可以启动系统

要注意的问题，/etc/fstab默认会挂在EFI分区，注释掉它

GRUB能识别读取mdraid的内容，/boot/grub文件夹可以放在软raid的设备上。

还有个问题，如果EFI启动项条目不识别，需要进BIOS，或者在liveCD中，使用efibootmgr编辑nvram，添加启动条目。


# 恢复问题

当一个盘，例如/dev/vda掉盘了，可以正常启动和使用，只是RAID降级了，如果修改内容后，两块盘不同步了，将硬盘插回来，或者新建一个硬盘后，他并不会自动同步。

如果他没有自动同步，那么，使用mdadm命令添加

```
mdadm --add /dev/md0 /dev/vda2
```

如果是新硬盘，应该先分好区，然后将vda2分区加入到软RAID里。


然后等待同步，同步进度可以通过/proc/mdstat看到


```
# cat /proc/mdstat
Personalities : [raid1] [linear] [multipath] [raid0] [raid6] [raid5] [raid4] [raid10]
md0 : active raid1 vda2[2] vdb2[1]
      31145984 blocks super 1.2 [2/1] [_U]
      [==>..................]  recovery = 12.2% (3823424/31145984) finish=2.6min speed=173792K/sec
```


在实验过程中遇上的一些困惑


## 1.mdraid设备在引导的哪个过程中被组装的？

grub有 `mdraid1x` 模块可以读取mdraid的设备，所以/boot/grub放在raid里也没关系，但是grubx64.efi需要单独放在一个EFI分区里，并且这个efi是包含了这些模块的，其他模块可以在/boot/grub里在载入，可以通过类似这样的命令制定模块包含在grub核心里

```
grub-install -v --modules="part_gpt part_msdos lvm mdraid09 mdraid1x" --efi-directory=/boot/efi --no-uefi-secure-boot --force-extra-removable --target=x86_64-efi
```

## 2.虽然grub能通过模块读取raid上的数据，但是，当载入linux后，内核如何读取raid上的数据？

当启动linux内核后，还会载入initrd,装载raid的脚本就包含在initramfs里，这些脚本不需要我们自己编写，在安装系统的时候，把mdadm安装了，再update-initramfs即可。

mdadm包有包含一部分打包到initramfs,还有个注意的地方，raid1使用到raid1.ko的，如果initramfs中没有包含这个ko,应该在 `/etc/initramfs-tools/modules` 中加入要打包到initramfs的内核模块。

## 3.掉盘后EFI分区挂载失败导致的启动失败

如果拿掉一个盘，会导致一个EFI分区不存在，挂载失败还是会阻塞开机，进入恢复模式，可以不要让efi分区开机自动挂载解决。

