title: "pixel3 解锁bootloader"
date: 2020-08-15T09:55:16+08:00
tags: ["Android"]
categories: ["Android"]
---

参考官方教程 https://source.android.com/setup/build/running  

## step1
a. 进入手机设置(setting)->关于手机(about phone)->点击版本号(Build number) 7次  
b. 当看到“你已经处于开发者模式”，点击返回。  
c. 系统(system)->高级->开发者选项(Developer options)->打开OEM锁和USB调试  

## step2
USB连接手机，确保ADB已配置好，并且可以和手机正常通信。若`adb deviuces`报`no permission`错误，则以下步骤处理:  
a. `lsusb` 找到手机的id，即`18d1:4ee7`。
```
Bus 001 Device 009: ID 18d1:4ee7 Google Inc. Pixel 3
```
b. `sudo nano /etc/udev/rules.d/51-android.rules` 编辑如下：

```
SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", ATTR{idProduct}=="4ee7", MODE="0666", GROUP="plugdev"
```
c. 重启电脑
d. `adb devices` 显示已链接。
```
List of devices attached
88PX01YWW	device
```

## step3

**注意,解锁会使手机内数据丢失，解锁前请备份数据**

使用`adb reboot bootloader`命令重启进入bootloader,待显示bootloader画面后

```sh
fastboot flashing unlock
```

此处有坑，我在执行此命令后一直在`<waiting for devices>`并没有丝毫反映。`fastboot devices` 显示`no permissions (user in plugdev group; are your udev rules wrong?)`。原因是bootloader模式下有不同的ID,此时`lsusb`查看ID,并按照之前的步骤编辑 51-android.rules文件，将此ID也添加进去，最终如下：

```
SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", ATTR{idProduct}=="4ee7", MODE="0666", GROUP="plugdev"
SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", ATTR{idProduct}=="4ee0", MODE="0666", GROUP="plugdev"
```

然后可能需要再次重启电脑，之后`fastboot flashing unlock`命令可以顺利执行了，执行后，手机会提示，使用音量上下选择是否解锁，并使用电源键选择解锁。开机后系统被重置了，需要重新设置。


## 重新锁bootloader

执行命令

```
fastboot flashing lock
```

