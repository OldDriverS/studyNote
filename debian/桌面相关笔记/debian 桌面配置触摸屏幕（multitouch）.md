# debian 桌面配置触摸屏幕（multitouch）

主机有3块屏幕：

- 一块1080P触摸屏
- 一块2k右侧辅屏
- 一块4k主屏幕

需要配置Xserver支持触摸屏在1080P屏幕上触摸，桌面环境不熟悉，配置起来比较困难，幸好，在arch wiki中，让我找到了这方面的资料。

涉及的软件包有，如果没安装，则需要安装。

- xinput 设置x系统中的输入

- x11-xserver-utils： xrandr所在的包，用来查询显示器。

- xinput-calibrator（可选），这个玩意是用来触摸屏校正的，这里并没有使用到

- xserver-xorg-input-multitouch： 这个是关键,他为触摸手势提供支持。

- xserver-xorg-input-evdev： evdev协议支持。

  

  

  ## 查询屏幕信息

  ```bash
  2700X@Debian:~$ xrandr 
  Screen 0: minimum 320 x 200, current 6400 x 3240, maximum 16384 x 16384
  DisplayPort-0 connected 1920x1080+1920+2160 (normal left inverted right x axis y axis) 344mm x 195mm
  ```

  我这里 **DisplayPort-0** 就是其中一个屏幕，复制他的名称，通过xinput让触摸屏范围限制在绑定的显示器。

  ## 查询触摸屏设备信息

  ```
  2700X@Debian:~$ xinput
  ⎡ Virtual core pointer                          id=2    [master pointer  (3)]
  ⎜   ↳ Virtual core XTEST pointer                id=4    [slave  pointer  (2)]
  ⎜   ↳ G2Touch Multi-Touch by G2TSP              id=8    [slave  pointer  (2)]
  .....
  ```

  我的触摸屏设备是 **G2Touch Multi-Touch by G2TSP** ，触摸屏设备的驱动是 `hid_multitouch` 模块，如果没有识别触摸屏，应该先检查驱动。

  

  

  ## 直接设置

  ```
  xinput --map-to-output 'G2Touch Multi-Touch by G2TSP' DisplayPort-0
  ```

  

  # 开启启动

  我尝试了很多方法，包括修改Xsetup、Xsession，最终配置都没有生效。。。我佛了，最终在桌面新增了一个快捷方式..需要用的时候，点一下就行了..
  
   

---

笔记资料来源：

[arch wiki Touchscreen](https://wiki.archlinux.org/title/Touchscreen)
