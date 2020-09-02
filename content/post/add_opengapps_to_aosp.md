---
title: "AOSP组入OpenGApps"
date: 2020-08-30T10:56:16+08:00
tags: ["Android"]
categories: ["Android"]
---


自己编译好的AOSP是不包括GMS(Google Mobile Services)的，不太方便玩耍，但是有Open GApps可以用。  
官网： https://opengapps.org/  
github: https://github.com/opengapps/aosp_build

## 环境： 

AOSP分支  ：android-10.0.0_r41  
lunch选项 ：aosp_blueline-userdebug  
实机      ：Pixel 3


组入步骤按以下走就行。

## 安装lunzip & git lfs

```sh
sudo apt-get install lunzip git-lfs
```

## 修改manifests

修改在AOSP源码根路径下的`.repo/manifest.xml`,在末尾前添加内容,添加后大概类似这样：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>

...

...

<remote name="opengapps" fetch="https://github.com/opengapps/"  />
  <remote name="opengapps-gitlab" fetch="https://gitlab.opengapps.org/opengapps/"  />

  <project path="vendor/opengapps/build" name="aosp_build" revision="master" remote="opengapps" />

  <project path="vendor/opengapps/sources/all" name="all" clone-depth="1" revision="master" remote="opengapps-gitlab" />

  <!-- arm64 depends on arm -->
  <project path="vendor/opengapps/sources/arm" name="arm" clone-depth="1" revision="master" remote="opengapps-gitlab" />
  <project path="vendor/opengapps/sources/arm64" name="arm64" clone-depth="1" revision="master" remote="opengapps-gitlab" />

  <project path="vendor/opengapps/sources/x86" name="x86" clone-depth="1" revision="master" remote="opengapps-gitlab" />
  <project path="vendor/opengapps/sources/x86_64" name="x86_64" clone-depth="1" revision="master" remote="opengapps-gitlab" />
  
  <repo-hooks in-project="platform/tools/repohooks" enabled-list="pre-upload" />
</manifest>
```

## 拉取

然后`repo sync`一下，
保险起见执行 `repo forall -c "git lfs pull"`

## 修改device.mk

由于我使用的是Pixel 3，所以我改的是device/google/crosshatch/aosp_blueline.mk,其他设备就找对应的mk文件修改，在末尾添加如下两行：

```makefile
GAPPS_VARIANT := stock
$(call inherit-product, vendor/opengapps/build/opengapps-packages.mk)
```
GAPPS_VARIANT 是[包类型](https://github.com/opengapps/opengapps/wiki/Package-Comparison)中的一个，根据自己需要选择。

## 编译、刷机


然后就`make -j6`等编译完成  

刷入手机中
```shell
adb reboot bootloader
fastboot flashall -w
```





