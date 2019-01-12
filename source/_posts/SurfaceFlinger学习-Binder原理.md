---
title: surfaceflinger学习--Binder原理
date: 2019-01-12 20:34:49
tags: 
- android_framework
cover: /images/ape_fwk_graphics.png
---

看到老罗android之旅分析android源码简直逆天，跟代码都跟到驱动层去了，看了半天连个大概都看不懂，还有《深入理解Android》，跟一段代码跟着跟着都不知道自己在看啥。发现还是要对系统框架有个大概的了解，再去看这些东西比较好。

SurfaceFlinger作为android绘制服务，涉及东西还是挺多的。

*   跨进程通信Binder机制
*   上层的View系统
*   下层的Display系统

首先是Binder，Binder主要用于Android中的跨进程通信，类似与socket一样的东西，由于Android将Bindder分为了业务层与传输层，导致了一堆Bindder对象的出现，再加上有一个比较特殊的服务ServiceManager一部分使用了Bindder，一部分又没有有使用Bindder，还有什么Server跟Service，导致看的整个人都不好。

先放一张IBinder的图
**BpBinder**与**BBinder**是一对，用于传输层的通信
**BpSerivce**与**BnSerivce**是一对，用于业务层的通信
**ISerivce**是客户端使用的Serivce接口
**Serivce**是具体的实现

最终的通信发生在IPCThreadState，具体实现使用/dev/binder这个设备文件及使用ioctl来与驱动通信。

---
举个荔枝

SurfaceFlinger与Ap通信是通过Binder来通信的，而且还不仅仅是一种，反正先关注ISurfaceComposerClient

除了个别名称不一致，其他都与之前分析的IBinder架构是一致的

---
分别来看一下其中每个类

### ISurfaceComposerClient
ISurfaceComposerClient在ISurfaceComposerClient.h中定义
具体内容就是几个接口的声明

```java
class ISurfaceComposerClient : public IInterface
{
public:
    DECLARE_META_INTERFACE(SurfaceComposerClient);
    ...
    virtual status_t createSurface(
        const String8& name, uint32_t w, uint32_t h,
        PixelFormat format, uint32_t flags,
        sp<IBinder>* handle,
        sp<IGraphicBufferProducer>* gbp) = 0;
    virtual status_t destroySurface(const sp<IBinder>& handle) = 0;
    virtual status_t clearLayerFrameStats(const sp<IBinder>& handle) const = 0;
    virtual status_t getLayerFrameStats(const sp<IBinder>& handle, FrameStats* outStats) const = 0;
};
```

### BpSurfaceComposerClient
BpSurfaceComposerClinet在ISurfaceComposerClient.cpp定义与实现
```cpp
class BpSurfaceComposerClient : public BpInterface<ISurfaceComposerClient>
{
public:
    BpSurfaceComposerClient(const sp<IBinder>& impl)
        : BpInterface<ISurfaceComposerClient>(impl) {
    }

    virtual status_t createSurface(const String8& name, uint32_t w,
            uint32_t h, PixelFormat format, uint32_t flags,
            sp<IBinder>* handle,
            sp<IGraphicBufferProducer>* gbp) {
        // 打包参数
        Parcel data, reply;
        data.writeInterfaceToken(ISurfaceComposerClient::getInterfaceDescriptor());
        data.writeString8(name);
        data.writeInt32(w);
        data.writeInt32(h);
        data.writeInt32(format);
        data.writeInt32(flags);

        // 用IBinder的transact，开始通信层的通信
        remote()->transact(CREATE_SURFACE, data, &reply);

        // 接收结果
        *handle = reply.readStrongBinder();
        *gbp = interface_cast<IGraphicBufferProducer>(reply.readStrongBinder());
        return reply.readInt32();
    }
    ...(其他在ISurfaceComposerClient定义的借口)
};
```

### BnSurfaceComposerClient
BnSurfaceComposerClient分别在ISurfaceComposerClient.h和ISurfaceComposerClient.cpp定义，因为服务端的Client类需要继承自BnSurfaceComposerClient，BnSurfaceComposerClient需要被导出。

BnSurfaceComposerClient的定义，只有一个onTransact函数
```cpp
class BnSurfaceComposerClient: public BnInterface<ISurfaceComposerClient> 
{ 
     public: virtual status_t onTransact(uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0); 
}; 
```

BnSurfaceComposerClient的实现
```cpp
status_t BnSurfaceComposerClient::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
     switch(code) {
        case CREATE_SURFACE: {
            CHECK_INTERFACE(ISurfaceComposerClient, data, reply);

            // 读取参数
            String8 name = data.readString8();
            uint32_t w = data.readInt32();
            uint32_t h = data.readInt32();
            PixelFormat format = data.readInt32();
            uint32_t flags = data.readInt32();
            sp<IBinder> handle;
            sp<IGraphicBufferProducer> gbp;

            // 这个是虚函数，运行时会调用子类具体实现的函数
            status_t result = createSurface(name, w, h, format, flags,
                    &handle, &gbp);

            // 打包返回值
            reply->writeStrongBinder(handle);
            reply->writeStrongBinder(gbp->asBinder());
            reply->writeInt32(result);
            return NO_ERROR;
        } break;
        ...（其他ISurfaceComposerClient中接口对应的信息处理)
    }
}
```

再看通信层的

### BpBinder
BpBinder在BpBinder.h与BpBinder.cpp 定义与实现 由于通信关键只跟transact有关，这里先只贴transact相关的代码

BpBinder的定义
```cpp
class BpBinder : public IBinder 
{ 
    public: 
    ...(一些binder状态查询函数) 
    virtual status_t transact( uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0);
    ...(一些binder连接管理函数)
    ...(一些锁机制)
};
```

BpBinder的实现
```cpp
status_t BpBinder::transact( uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) 
{ 
    // Once a binder has died, it will never come back to life. 
    if (mAlive) { 
        // 这里可以看到将工作委托给了IPCThreadState 
        status_t status = IPCThreadState::self()->transact( mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0; 
        return status; 
    }
    return DEAD_OBJECT;
}
```

### BBinder
BBinder定义
```java
class BBinder : public IBinder
{
public:
    ...(一些binder状态查询函数) 
    // 这个函数会被IPCThreadState所调用，并调用onTransact函数
    virtual status_t    transact(   uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
    ...(一些binder的操作)
protected:
    // 会被子类实现所覆盖
    virtual status_t    onTransact( uint32_t code,
                                    const Parcel& data,
                                    Parcel* reply,
                                    uint32_t flags = 0);
    ...(一些乱七八糟的东西)
};
```

BBinder实现，主要是transact函数
```java
status_t BBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    data.setDataPosition(0);
    status_t err = NO_ERROR;
    switch (code) {
        ...（其他情况)
        default:
            // 将会调用子类的onTransact
            err = onTransact(code, data, reply, flags);
            break;
    }
    ...
}
```

Binder层的通信差不多就这样，按这种思路的话下层应该有一个类似listener的东西，当BpBinder做transact的时候，会通知并调用到BBinder的onTransact函数。

---
IPCThreadState这一层应该是通信层的真正的实现，涉及到线程与驱动

简单起见，只分析跟transact有关的代码

### IPCThreadState—客户端
```cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
        ...(参数检查及打印日志)
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
        ...(各种乱七八糟的东西)
        err = waitForResponse(reply);
        ...(参数检查及打印日志)

    return err;
}
```

writeTransactionData仅仅是填充数据到mOut
```cpp
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;
    ...(初始化tr)
    ...(根据data填充tr)
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```

waitForResponse发起了请求并等待驱动返回
```cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    int32_t cmd;
    while (1) {
        talkWithDriver();
        ...(检查返回)
        cmd = mIn.readInt32();

        switch (cmd) {
        ...(各种错误情况处理处理)
        case BR_REPLY:
        ...(各种对reply的处理)
        break;
        default:
            err = executeCommand(cmd);  // ！！这里是服务端使用的
            break;
        }
    }

    ...(错误处理）
    return err;
}
```

talkWithDriver与驱动通信
```cpp
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    ...(初始化bwr)
    ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr);
    ...(错误处理)
    return err;
}
```

### IPCThreadState—-服务端
服务端在注册服务的时候会调用IPCThreadState的两个函数—-startThreadPool与joinThreadPool,应该就是listener的角色吧
startThreadPool调到最后还是会调joinThreadPool，所以直接看joinThreadPool
joinThreadPool在IPCThreadState中实现，相当于时刻等待驱动的消息
```cpp
void IPCThreadState::joinThreadPool(bool isMain)
{
    ...(处理一些东西）
    status_t result;
    do {
        ...(处理一些东西）
        result = getAndExecuteCommand();
        ...(处理一些东西）
    } while (result != -ECONNREFUSED && result != -EBADF);
    ...(处理一些东西）
}
```

getAndExecuteCommand获取从驱动过来的消息并执行相应的命令
```cpp
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;
    result = talkWithDriver();
    ...(各种判断)
    result = executeCommand(cmd);
    ...(各种处理)
    return result;
}
```

executeCommand用于执行从客户端过来的命令
```cpp
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    switch (cmd) {
    ...(各种其他操作)
    case BR_TRANSACTION:
        {
            ...(各种对tr的操作)              
            Parcel reply;
            sp<BBinder> b((BBinder*)tr.cookie);
            b->transact(tr.code, buffer, &reply, tr.flags);

            ...(设置reply)
            sendReply(reply, 0);
            ...(各种其他操作) 
        }
        break;
    ...(各种其他操作)
    }
    ...(各种错误处理)
}
```