---
title: "网易云音乐在高分辨率下缩放问题"
date: 2020-09-12T16:42:00+08:00
---

Linux下没几个好用的网络音乐播放器，网抑云算良心了，但是在高分屏下字体太小，根据几个博文修改`--force-device-scale-factor=2`后字体是正常了，但是界面依然缩成一坨。  

最近换了电脑后重装又遇到了这个问题,记得是之前一番搜索后从一篇博文中看到了解决方法，现在也没再找到。只好翻出旧电脑，找出配置。修改/usr/share/applications/netease-cloud-music.desktop如下：

```
Exec=env QT_SCALE_FACTOR=1.6 netease-cloud-music %U
```
记录一下，省的再找不到。