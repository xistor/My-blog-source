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

## 遇到的问题

1. 刷完之后遇到了这个问题,显示“Android Setup keeps stopping”

![error](/img/add_opengapps_to_aosp/error.png)

打开ADB,看log发现是权限问题

```
WifiService: Permission violation - getScanResults not allowed for uid=10056,
packageName=com.google.android.setupwizard, reason=java.lang.SecurityException: 
UID 10056 has no location permission

...

```

解决方法：在vendor/opengapps/build/modules/SetupWizard/Android.mk中添加一句，给SetupWizard加上platform签名
```
LOCAL_CERTIFICATE := platform
```
加完这个需要`make clean`，重新编译才能生效。


2. 上面问题解决后，再次刷人发现PixelLauncher也在挂，依然是权限问题，报错类似这样

```
03-25 18:04:17.889 750 7019 W ActivityManager: Permission Denial: setShelfHeight() from pid=8517, uid=10012 requires android.permission.STATUS_BAR
03-25 18:04:17.890 8517 8517 D AndroidRuntime: Shutting down VM
03-25 18:04:17.891 8517 8517 E AndroidRuntime: FATAL EXCEPTION: main
03-25 18:04:17.891 8517 8517 E AndroidRuntime: Process: com.google.android.apps.nexuslauncher, PID: 8517
03-25 18:04:17.891 8517 8517 E AndroidRuntime: java.lang.RuntimeException: Unable to resume activity
```

由于PixelLauncher貌似没有申请`android.permission.STATUS_BAR`权限，所以给他重新签名也没效果，这个问题找到两种解决方式
- 如果使用的是userdebug版本的话，可以执行`adb push packages.xml /data/system/packages.xml`，将package.xml拉到本地修改，在`com.google.android.apps.nexuslauncher`下的`perm`内添加下面两项, 然后再`adb push`回原位置。

```xml
<item name="android.permission.STATUS_BAR" granted="true" flags="0"/>
<item name="android.permission.MANAGE_ACTIVITY_STACKS" granted="true" flags="0" />
```

- 或者[参考](https://c55jeremy-tech.blogspot.com/2019/04/aosppixel-2-romrom.html)修改frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
删除权限检查的地方

```diff
--- a/services/core/java/com/android/server/wm/WindowManagerService.java
+++ b/services/core/java/com/android/server/wm/WindowManagerService.java
@@ -6001,8 +6001,8 @@ public class WindowManagerService extends IWindowManager.Stub
 
     @Override
     public void setShelfHeight(boolean visible, int shelfHeight) {
-        mAmInternal.enforceCallerIsRecentsOrHasPermission(android.Manifest.permission.STATUS_BAR,
-                "setShelfHeight()");
+        //mAmInternal.enforceCallerIsRecentsOrHasPermission(android.Manifest.permission.STATUS_BAR,
+        //        "setShelfHeight()");
         synchronized (mWindowMap) {
             getDefaultDisplayContentLocked().getPinnedStackController().setAdjustedForShelf(visible,
                     shelfHeight)

```

