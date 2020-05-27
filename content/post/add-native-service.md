---
title: "Android 使用AIDL添加 native service"
date: 2020-05-27T23:03:16+08:00
tags: ["android"]
categories: ["android"]
---

### 1. 下载 AOSP源码，配置编译环境
略
### 2. 目录结构
device下新建一个sample目录保存我们即将写的native服务,我的目录结构如下，这个service将在sample目录中实现。

        device
            |
            | -xx
                |-service
                    |-sample
                        |-aidl
                        |-include
                        |-process
                        |-service
                        |-test
                        Android.mk

### 3. 编写AIDL文件

新建AIDL文件aidl,声明接口，接口名称要与文件名一致。 in/out来表示是输入/输出参数，默认是in，out参数在最终生成的从C++源文件会以指针实现。这里定义了一个DoSomething接口，有两个参数:n和一个内含string的vector，功能是返回n个"Hello"在ouput数组中。

```aidl
package sample;

interface ISample {
void DoSomething(int n, out List<String> output);
}
```

新建一个Android.mk文件,编译生成对应的lib库以及头文件。
```makefile
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:=\
            sample/ISample.aidl \

LOCAL_MODULE_TAGS := optional

LOCAL_C_INCLUDES :=             \
            $(TOP)/frameworks/native/include \
            $(TOP)/system/core/base/include \

LOCAL_SHARED_LIBRARIES := \
            libbinder \
            libcutils \
            liblog \
            libutils \

LOCAL_MODULE:=libsample-aidl


include $(BUILD_SHARED_LIBRARY)
```
编译会生成BnSample.h,BpSample.h和ISample.h三个头文件，在out/target/product/generic_x86_64/obj/SHARED_LIBRARIES/libsample-aidl_intermediates/aidl-generated 目录下。同级的src目录下还有对应的cpp源文件ISample.cpp,AIDL帮我们实现好了这些格式化的文件，自己手写实现也是可以的。

### 4. 实现 sampleService
sampleService.h, sampleService继承自BnSample。
```cpp
#ifndef SAMPLE_SERVICE_H
#define SAMPLE_SERVICE_H

#include<BnSample.h>
#include <vector>

namespace android {

class sampleService : public sample::BnSample {
public:

    sampleService();
    ~sampleService();
    virtual ::android::binder::Status DoSomething(int32_t n, ::std::vector<::android::String16>* output);
}; 

}

#endif  // SAMPLE_SERVICE_H

```

sampleService.cpp，在此实现DoSomething的逻辑，这里就简单的把count个"Hello"塞到vector中返回。

```cpp
#include"sampleService.h"

using namespace android;

sampleService::sampleService() {

}

sampleService::~sampleService() {
    
}

::android::binder::Status sampleService::DoSomething(int32_t n, ::std::vector<::android::String16>* output) {

    for(int i = 0; i < n; i++) {
        output->push_back(String16("Hello"));
    }

    return ::android::binder::Status::ok(); 
}
```
    

service的Android.mk 如下：

```makefile
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:=\
            sampleService.cpp \

LOCAL_MODULE_TAGS := optional

LOCAL_C_INCLUDES :=             \
            $(TOP)/frameworks/native/include \
            $(TOP)/system/core/base/include \
            $(TOP)/device/x/eva0/framerworks/service/sample/include \
            $(TARGET_OUT_INTERMEDIATES)/SHARED_LIBRARIES/libsample-aidl_intermediates/aidl-generated/include/sample

LOCAL_SHARED_LIBRARIES := \
            libbinder \
            libcutils \
            liblog \
            libutils \
            libsample-aidl \

LOCAL_MODULE:=libsampleservice

include $(BUILD_SHARED_LIBRARY)

```

然后我们需要一个进程来承载我们的service，在process目录下新建sample_main.cpp文件，在main函数中调用serviceManager的addService方法将我们的service注册进去，以便客户端可以找到。

```cpp
#include <sys/types.h>
#include <binder/IPCThreadState.h>
#include <binder/ProcessState.h>
#include <binder/IServiceManager.h>
#include <cutils/log.h>
#include "sampleService.h"

using namespace android;

int main(int argc, char** argv) {

    sp<ProcessState> proc(ProcessState::self());

    sp<IServiceManager> sm = defaultServiceManager();

    ALOGI("ServiceManager: %p", sm.get());

    sm->addService(String16("sampleService"), new sampleService());

    ProcessState::self()->startThreadPool();

    IPCThreadState::self()->joinThreadPool();

    return 0;

}
```

对应的的mk如下，编译生成的sampleService就是我们service的可执行文件。

```makefile
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
 
LOCAL_SRC_FILES:=\
			sample_main.cpp \
 
LOCAL_MODULE_TAGS := optional
 
LOCAL_C_INCLUDES :=             \
			$(TOP)/frameworks/native/include \
			$(TOP)/system/core/base/include \
			$(TOP)/device/x/eva0/framerworks/service/sample/include \
			$(TARGET_OUT_INTERMEDIATES)/SHARED_LIBRARIES/libsample-aidl_intermediates/aidl-generated/include/sample

LOCAL_SHARED_LIBRARIES := \
			libbinder \
			libcutils \
			liblog \
			libutils \
			libsample-aidl \
			libsampleservice \

LOCAL_MODULE:= sampleService
 
include $(BUILD_EXECUTABLE)	
	
```

### 6. 实现客户端

客户端就是通过serviceManager取得sampleService的指针后，简单的调用下DoSomething接口,count值传入3，并把返回的vector打印出来。

```cpp
#include <binder/IServiceManager.h>
#include <binder/Parcel.h>
#include <utils/Errors.h>
#include <stdio.h>
#include <String16.h>
#include <String8.h>
#include <vector>
#include <iostream>
#include "sampleService.h"

using namespace android;

int main () {

    sp<sample::ISample> sample_srv = interface_cast<sample::ISample>(defaultServiceManager()->getService(String16("sampleService")));

    std::vector<String16> hellos;
    sample_srv->DoSomething(3, &hellos);

    for (String16 s:hellos ) {
        std::cout << String8(s.string()).string() << std::endl;
    }

    return 0;
}

```

客户端的mk文件
```makefile
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:=\
            sample_test.cpp \

LOCAL_MODULE_TAGS := optional

LOCAL_C_INCLUDES :=             \
            $(TOP)/frameworks/native/include \
            $(TOP)/system/core/base/include \
            $(TOP)/device/x/service/sample/include \
            $(TOP)/system/core/include/utils \
            $(TOP)/device/x/eva0/framerworks/service/sample/include \
            $(TARGET_OUT_INTERMEDIATES)/SHARED_LIBRARIES/libsample-aidl_intermediates/aidl-generated/include/sample

LOCAL_SHARED_LIBRARIES := \
            libbinder \
            libcutils \
            liblog \
            libutils \
            libsample-aidl \
            libsampleservice \

LOCAL_MODULE:= sampletest

include $(BUILD_EXECUTABLE)	
```
### 7. 测试
lunch之后输入emulator启动模拟器，将sampleService和Sampletest push进模拟器。
```sh
sampleService &
sampletest
```
我们客户端传入的参数是3，所以输出三行Hello就代表成功了。


### 8. 参考
https://android.googlesource.com/platform/system/tools/aidl/+/brillo-m10-dev/docs/aidl-cpp.md