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





