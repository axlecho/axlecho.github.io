---
title: 在framework添加system service
date: 2019-01-22 11:18:30
categories: Android
tags:
    - android
    - framework
---

在Context.java添加服务名称
```java
//--->frameworks/base/core/java/android/content/Context.java
public static final String THREEKEY_SERVICE = "threekey";

@StringDef {
    ...
    THREEKEY_SERVICE
}
```

在SystemServicer注册服务
```java
//--->frameworks/base/services/java/com/android/server/SystemServer.java
private void startOtherServices() {
    ...
    ServiceManager.addService("ThreeKeyService", new ThreeKeyService);
    ...
}
```

在ContextImpl.java添加获取服务管理接口
```java
//--->frameworks/base/core/java/android/app/ContextImpl.java
static {
    ....
    registerService(Context.THREEKEY_SERVICE,ThreeKeyManager.class,
            new CachedServiceFetcher<ThreeKeyManager>() {
        @Override
        public ThreeKeyManager createService(ContextImpl ctx) {
            return new ThreeKeyManager(ctx);
        }});
}
```

Service的aidl
```java
interface IThreeKeyExService {
      
}
```

Service 本体
```java
public class ThreeKeyService extends IThreeKeyService.Stub {

}
```

Manager
```java
public class ThreeKeyManager {
    public ThreeKeyManager(Context context) {

    }

    static public IThreeKeyService getService()
    {
        IBinder b = ServiceManager.getService("notification");
        return IThreeKeyService.Stub.asInterface(b);
    }

}
```