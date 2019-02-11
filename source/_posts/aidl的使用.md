---
title: aidl的使用
date: 2019-01-22 11:23:23
tags: android
---

接口文件aidl
```java
package com.zst.xposed.halo.floatingwindow3.services;

interface IActivityManagerService {
    void snapActivityTop(int id);
    void snapActivityBottom(int id);
}
```

manager
```java
package com.zst.xposed.halo.floatingwindow3.services;
import com.zst.xposed.halo.floatingwindow3.IActivityManagerService;
public class ActivitManager extends IActivityManagerService.Stub {

    public ActivityManagerService mService;
    public ActivitManager(ActivityManagerService service) {
        mService = service;
    }

    public void snapActivityTop(int id) {}
    public void snapActivityBottom(int id) {}
}
```

service
```java
package com.zst.xposed.halo.floatingwindow3.services;

import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
import android.support.annotation.Nullable;

public class ActivityManagerService extends Service {

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return new ActivitManager(this);
    }
}
```