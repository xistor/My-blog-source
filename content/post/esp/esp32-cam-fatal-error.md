---
title: "Failed to connect to ESP32: Timed out waiting for packet header"
date: 2021-08-31T21:56:16+08:00
tags: ["esp32"]
categories: ["esp"]
---

20几块淘宝淘了一块Esp32-Cam, 网上也有很多配置教程， 环境配置没有什么大的问题，但在烧写固件的时候遇到了下面的错误：  

```

Looking for upload port...
Auto-detected: /dev/ttyUSB0
Uploading .pio/build/nodemcu-32s/firmware.bin
esptool.py v3.1
Serial port /dev/ttyUSB0
Connecting........_____....._____....._____....._____....._____....._____....._____

A fatal error occurred: Failed to connect to ESP32: Timed out waiting for packet header
*** [upload] Error 2

```

期间尝试了如下几种解决方法,都无果：  

1. 串口板从 CH340换成PL2303
2. 从PlatformIO换成Arduino IDE
3. 串口波特率改成115200
4. 5v供电和3.3v供电都尝试过。
5. 确认GPIO0已接GND
6. Connecting时按rst
7. 3.3v口加电容（这个还没试）

就在我无计可施时，无奈把杜邦线的GND换了个插...坑爹啊...  

网上几乎所有的教程都用了下图这种连接方式，我也是按照这个连的。

{{< figure src="/img/esp32-cam/esp32-cam.png"  class="center" title="esp32-cam 下载 ✘" width="500">}}

但是我这样连接是不能下载固件的，只有把usb转TTL的GND和esp32-cam的5v旁边的GND连接才可以下载（如下图），可能是因为这个模块是个寨版吧..上边都没有AI thinker的标志。

{{< figure src="/img/esp32-cam/esp32-cam-mod.png"  class="center" title="esp32-cam 下载 ✔" width="500">}}

总之，固件烧写这关算是过了。