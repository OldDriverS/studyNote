# Ryzen APU驱动安装笔记

## 故障情况

Debian 10 stable默认内核版本为4.19，但刚安装完系统启动，并没有加载驱动模块`amdgpu`,显卡驱动工作不正常，xserver也没法正常工作，没法启动桌面，安装完毕后，进入终端界面。


## 初步确定系统情况

确定内核版本号

```bash
$ uname -r
```

若是`debian10 stable版本` (代号buster)，内核版本为4.19，则有两种选择：

- 把整个系统整体升级至`testing` (代号bullseye)版本以获得更新版本的内核。(5.9)

- 在`stable`版本下添加`backports`源，`backports`源下有较新的候选版本软件包，有多个版本的内核可供选择。

这里记录以添加`backports`源作为处理方法。

## 换源

使用`sudo apt edit-sources`选中`nano`编辑`/etc/apt/sources.list`文件

```bash
$ sudo apt edit-sources

Select an editor.  To change later, run 'select-editor'.
  1. /usr/bin/vim.nox
  2. /bin/nano        <---- easiest
  3. /usr/bin/vim.basic
  4. /usr/bin/mcedit
  5. /usr/bin/vim.tiny

Choose 1-5 [2]: 2
```

`source.list` 使用#注释国外源(主要为debian.org以及security相关的源)，新增国内镜像。如清华源、163，阿里，此处使用清华源。(ctrl+o保存、ctrl+x退出nano编辑器)，新增内容:

```
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster main non-free contrib
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ buster main non-free contrib
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security/ buster/updates main non-free contrib

#buster backports源
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ buster-backports  main non-free contrib
```

包管理操作参考debian参考手册，以下记录操作

```bash
#更新软件源索引
$ sudo apt update
```

## 换内核以及安装firmware固件包

debian 内核相关的软件包名称为`linux-image-xxxxx`，可以通过`apt search linux-image`模糊搜索候选的软件包，选取一个`5.x`版本，

```bash
$ sudo apt search linux-image
```

backports源下可能有多个候选内核，包名类似

```
linux-image-内核版本号-amd64
linux-image-内核版本号-amd64-dbg
linux-image-内核版本号-cloud-amd64
linux-image-内核版本号.bpo.xx-amd64
...
```

`bpo` 是backports源软件包

`amd64`是可选的架构，用于64位PC（X86_64）

`rt` 打了PREEMPT_RT内核实时补丁的版本，这个版本为我们也用不上

`cloud` 用于Amazon EC2, Google Compute Engine 、 Microsoft Azure cloud，这个版本我们不需要。

`dbg` 这个包用于调试，不是内核本身，只是符号相关的文件，我们也用不上。

选取一个5.x版本的内核，仅有`amd64`后缀的，没有`dbg`后缀的软件包安装。

显卡、无线网卡、部分网卡，不仅需要内核模块，还需要加载firmware才能正常工作,安装firmware软件包：

```bash
$ sudo apt install firmware-linux-nonfree
```

> 如果是很新的平台，例如刚发布的apu，可能debian的固件中并未包含新硬件相关固件，需要去同步`linux-firmware.git`仓库，获取更新一些的firmware安装到本机。[北外语镜像站同步的linux-firmware.git](https://mirrors.bfsu.edu.cn/help/linux-firmware.git/), [linux-firmware.git](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git)，根据需要使用。

安装固件和新内核完毕后重启，开机后在grub菜单中选择新内核启动，使用`uname -r` 验证运行的内核版本。


## 确认amdgpu驱动工作正常

根据 `dmesg` 筛选错误信息，进一步查看是否存在其他错误:

```bash
$ sudo dmesg -l err

[    5.136341] pci 0000:00:00.2: AMD-Vi: Unable to read/write to IOMMU perf counter.
```

以上IOMMU的错误，不影响使用。未发现firmware相关错误,IOMMU问题可通过修改内核参数添加`iommu=soft`，此处并不是问题重点，忽略。

`demsg` 筛选 `amdgpu` 关键字查找模块日志输出，模块无异常日志。

```
$ sudo dmesg | grep -i "amdgpu"

[    6.922236] [drm] amdgpu kernel modesetting enabled.
[    6.976424] amdgpu: Topology: Add APU node [0x0:0x0]
[    6.982614] fb0: switching to amdgpudrmfb from EFI VGA
[    6.994830] amdgpu 0000:08:00.0: vgaarb: deactivate vga console
[    7.010220] amdgpu 0000:08:00.0: amdgpu: Trusted Memory Zone (TMZ) feature disabled as experimental (default)
[    7.078110] amdgpu 0000:08:00.0: firmware: direct-loading firmware amdgpu/picasso_gpu_info.bin
[    7.132712] amdgpu: ATOM BIOS: 113-PICASSO-117
[    7.137496] amdgpu 0000:08:00.0: firmware: direct-loading firmware amdgpu/picasso_sdma.bin
[    7.170158] amdgpu 0000:08:00.0: amdgpu: VRAM: 1024M 0x000000F400000000 - 0x000000F43FFFFFFF (1024M used)
[    7.180414] amdgpu 0000:08:00.0: amdgpu: GART: 1024M 0x0000000000000000 - 0x000000003FFFFFFF
[    7.189492] amdgpu 0000:08:00.0: amdgpu: AGP: 267419648M 0x000000F800000000 - 0x0000FFFFFFFFFFFF
[    7.343916] [drm] amdgpu: 1024M of VRAM memory ready
[    7.349303] [drm] amdgpu: 3072M of GTT memory ready.
[    7.369210] amdgpu 0000:08:00.0: firmware: direct-loading firmware amdgpu/picasso_asd.bin
[    7.383826] amdgpu 0000:08:00.0: firmware: direct-loading firmware amdgpu/picasso_ta.bin
[    7.396645] amdgpu 0000:08:00.0: firmware: direct-loading firmware amdgpu/picasso_pfp.bin

....此处省略一堆日志

[    8.277667] amdgpu 0000:08:00.0: amdgpu: ring vcn_dec uses VM inv eng 1 on hub 1
[    8.285640] amdgpu 0000:08:00.0: amdgpu: ring vcn_enc0 uses VM inv eng 4 on hub 1
[    8.293748] amdgpu 0000:08:00.0: amdgpu: ring vcn_enc1 uses VM inv eng 5 on hub 1
[    8.301848] amdgpu 0000:08:00.0: amdgpu: ring jpeg_dec uses VM inv eng 6 on hub 1
[    8.321136] [drm] Initialized amdgpu 3.39.0 20150101 for 0000:08:00.0 on minor 0
```

 `lspci` 筛选查看pci设备详情，存在 `Kernel modules: amdgpu` 字段，指示设备使用的驱动为amdgpu。

```
$ lspci -kv -s $(lspci | awk '/VGA/{print $1}')
08:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Picasso (rev c9) (prog-if 00 [VGA controller])
        Subsystem: ASUSTeK Computer Inc. Picasso
        Flags: bus master, fast devsel, latency 0, IRQ 62, IOMMU group 12
        Memory at e0000000 (64-bit, prefetchable) [size=256M]
        Memory at f0000000 (64-bit, prefetchable) [size=2M]
        I/O ports at e000 [size=256]
        Memory at fcb00000 (32-bit, non-prefetchable) [size=512K]
        Expansion ROM at 000c0000 [virtual] [disabled] [size=128K]
        Capabilities: <access denied>
        Kernel driver in use: amdgpu
        Kernel modules: amdgpu
```

如上述均正常，但Xserver还是未能正常工作，驱动问题已排除，需要考虑其他问题，例如x服务、桌面没有安装，显示管理服务没有安装之类。

检查 `xserver-xorg` 有没安装,处理一下

```
$ sudo apt install xserver-xorg
```

若xserver已经安装，则xserver的日志在 `/var/log/Xorg.0.log` ，查看日志中错误信息，再根据里面的线索进一步排查，日志信息很多，需要筛选有价值的错误信息。

可以使用`less`命令配合正则搜索定位查找日志文件

```bash
$ sudo less /var/log/Xorg.0.log
```

## 安装显示管理器(Dispaly Manager/DM)和桌面(Desktop)

如果没有安装桌面环境，需要手动安装一个，此处选择了 `mate` 桌面 ， DM使用 `lightdm`

```bash
$ sudo apt install mate-desktop-environment lightdm
``` 

`debian 10` 使用systemd作为init管理系统，重启lightdm服务

```bash
# 允许lightdm开启启动
$ sudo systemctl enable lightdm
# 启动lightdm服务
$ sudo systemctl start lightdm
# 重启lightdm服务
$ sudo systemctl restart lightdm
```

## mesa驱动（视频解码和渲染相关）

让APU通过VAAPI使用核显硬件处理图形相关内容，提高效率，避免cpu太多压力。

ffmpeg、vlc播放器可能用到这方面内容。

```bash
sudo apt install mesa-va-drivers mesa-vdpau-drivers mesa-vulkan-drivers  mesa-utils 
```

- `mesa-utils` 是 glxinfo相关工具的软件包

- `mesa-va-drivers` vaapi相关，渲染设备为`/dev/dri/renderD128`

- 剩下两个是`VDPAU` `Vulkan` 驱动

安装、使用`vainfo`查看vaapi支持情况。

```bash
$ sudo apt install vainfo
```

查看信息（此处内核版本为5.9.0-5-amd64，mesa版本为20.2.6）

```bash
$ sudo vainfo
error: XDG_RUNTIME_DIR not set in the environment.
error: can't connect to X server!
libva info: VA-API version 1.10.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/radeonsi_drv_video.so
libva info: Found init function __vaDriverInit_1_10
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.10 (libva 2.9.0)
vainfo: Driver version: Mesa Gallium driver 20.2.6 for AMD Radeon(TM) Vega 8 Graphics (RAVEN, DRM 3.39.0, 5.9.0-5-amd64, LLVM 11.0.0)
vainfo: Supported profile and entrypoints
      VAProfileMPEG2Simple            : VAEntrypointVLD
      VAProfileMPEG2Main              : VAEntrypointVLD
      VAProfileVC1Simple              : VAEntrypointVLD
      VAProfileVC1Main                : VAEntrypointVLD
      VAProfileVC1Advanced            : VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSlice
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointEncSlice
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointVLD
      VAProfileVP9Profile2            : VAEntrypointVLD
      VAProfileNone                   : VAEntrypointVideoProc

```


## AMD GPU 核显管理工具 radeontop

radeontop可以用于查看amd gpu核显工作状态，如主频、内存频率、使用率等。

[radeontop项目地址](https://github.com/clbr/radeontop)

根据github上的项目帮助说明,拉取并编译radeontop即可。

## AMDGPU 核显频率管理

相关参数位于 `/sys/class/drm/card0/device` ，此处只有一个显卡， `card0`。

amdgpu相关的文档位于源码目录：

[Documentation/gpu/amdgpu.rst](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/gpu/amdgpu.rst?h=linux-5.4.y)

[Documentation/gpu/amdgpu-dc.rst](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/Documentation/gpu/amdgpu-dc.rst?h=linux-5.4.y)

鸡肠子什么，懂是不可能懂的，这辈子都不可能懂的，机翻凑合看。可根据 `/sys/class/drm/card0/device` 下文件名查找相关注释参数说明，再进行调节，未知参数不应胡乱设置，

如需gpu降频，则在 `/sys/class/drm/card0/device/power_dpm_force_performance_level` 参数，

说明位于 [drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/drivers/gpu/drm/amd/amdgpu/amdgpu_pm.c?h=linux-5.4.y)

取值可能是 `auto` `low` `high` `manual` `profile_standard` ...等，更多要去看具体的说明。

设置为 `low` 则使用节能策略(需要切换至root用户)

```bash
$ sudo su
# echo low > /sys/class/drm/card0/device/power_dpm_force_performance_level
```

通过 `radeontop` 查看GPU节能模式下频率。Ryzen 3200g的是200Mhz。

```
0.20G / 1.25G Shader Clock  16.00% 
```

---

# 参考资料：

[debian使用手册：apt/aptitude包管理](https://www.debian.org/doc/manuals/debian-reference/ch02.zh-cn.html#_basic_package_management_operations)

[debian安装手册：需要固件的设备](https://www.debian.org/releases/stable/amd64/ch02s02.zh-cn.html)

[systemd参考手册(金步国译)](http://www.jinbuguo.com/systemd/systemctl.html#)

[arch wiki AMDGPU](https://wiki.archlinux.org/index.php/AMDGPU_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

[debian管理员手册：配置X11](https://debian-handbook.info/browse/zh-CN/stable/workstation.html#sect.x11-server-configuration)

[linux kernel 5.4.y 源码仓库](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/?h=linux-5.4.y)

[FFMPEG 硬件解码wiki](https://trac.ffmpeg.org/wiki/Hardware/VAAPI)