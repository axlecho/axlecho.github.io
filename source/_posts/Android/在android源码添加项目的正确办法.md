---
title: 在android源码添加项目的正确办法
date: 2019-01-19 22:34:25
tags: android_framework
---

源码添加项目的正确办法
2018-04-27
主要是Android.mk文件

lib版本

```shell
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE := your-module-name
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_CLASS := JAVA_LIBRARIES
LOCAL_SRC_FILES := $(call all-subdir-java-files)

LOCAL_JAVA_LIBRARIES := framework services
#LOCAL_SDK_VERSION := current  
include $(BUILD_JAVA_LIBRARY)
```

apk版本
```shell
# Android threekey
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)

LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(call all-java-files-under, src)
LOCAL_SDK_VERSION := current
LOCAL_PACKAGE_NAME := threekey-test
LOCAL_CERTIFICATE := platform
include $(BUILD_PACKAGE)
```

#### 路径问题

递归所有目录
```shell
LOCAL_SRC_FILES := $(call all-subdir-java-files)
```

指定目录
```
LOCAL_SRC_FILES := $(call $(src_dirs))
```

当前目录
```
LOCAL_SRC_FILES := $(call all-java-files-under, src)
```

#### 注册项目
具体看[google README.txt](https://android.googlesource.com/platform/vendor/sample/+/acd7c6b02a14e8694d0dced56ea37e79707fba1e/frameworks/PlatformLibrary/README.txt)
