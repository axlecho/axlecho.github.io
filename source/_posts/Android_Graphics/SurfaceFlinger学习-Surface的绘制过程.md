---
title: SurfaceFlinger学习--Surface的绘制过程
date: 2019-01-12 20:22:38
categories: Android_Graphics
tags: 
    - framework
    - graphics
---

![cover](/images/ape_fwk_graphics.png)
今天看到一篇很不错的的关于SurfaceFlinger的文章，主要是看到android源码中有一个简单明了的test，而且还被我编译过了，都不知道的前几个星期看的都是些什么东西- -
[Android 4.4(KitKat)中的设计模式-Graphics子系统](http://blog.csdn.net/jinzhuojun/article/details/17427491)

还是从test开始看起吧
test的在工程中的位置:frameworks/native/services/surfaceflinger/tests/resize,用于在屏幕上显示一块色块

```cpp
int main(int argc, char** argv)
{
    // set up the thread-pool
    sp<ProcessState> proc(ProcessState::self());
    ProcessState::self()->startThreadPool();

    // 连接到surfaceflinger
    sp<SurfaceComposerClient> client = new SurfaceComposerClient();

    // 创建surface
    sp<SurfaceControl> surfaceControl = client->createSurface(String8("resize"),600, 800, PIXEL_FORMAT_RGB_565, 0);
    sp<Surface> surface = surfaceControl->getSurface();

    // 设置Layer的z轴
    SurfaceComposerClient::openGlobalTransaction();
    surfaceControl->setLayer(100000);
    SurfaceComposerClient::closeGlobalTransaction();

    // 操纵Buffer中内容
    ANativeWindow_Buffer outBuffer;
    surface->lock(&outBuffer, NULL);
    ssize_t bpr = outBuffer.stride * bytesPerPixel(outBuffer.format);
    android_memset16((uint16_t*)outBuffer.bits, 0xffff, bpr*outBuffer.height);
    surface->unlockAndPost();

    IPCThreadState::self()->joinThreadPool();
    return 0;
}
```

连接到surfaceflinger
---
补充一点，ComposerSerivce创建时会调用getSerivce函数，然后再调用defaultServiceManager获取ServiceManager接口IServiceManager，再通过IServiceManager的getService获取到SurfaceFlinger服务的接口，ISurfaceComposer其实就是SurfaceFlinger的代理接口。

创建surface
---
补充：Layer继承至SurfaceFlingerConsumer::ContentsChangedListener,实现了Listener的几个处理函数

*   onFrameAvailable(const BufferItem& item)
*   onFrameReplaced(const BufferItem& item)
*   onSidebandStreamChanged()

设置Layer的z轴
---
使用openGlobaleTransaction可以将几个事务合并成一个提交,而且是必须的:

*All composer parameters must be changed within a transaction
several surfaces can be updated in one transaction, all changes are
committed at once when the transaction is closed.
closeGlobalTransaction() requires an IPC with the server.*

setLayer调用了SurfaceComposer的setLayer,与SurfaceFlinger的**连接有点问题**

操纵Buffer中的内容
---

其中BufferQueueProducer在queuebuffer中调用了BufferQueueCore的frameAvailableListener的onFrameAvailable方法，经过了一长串的调用最终会调用到Layer的onFrameAvailable。

下面是分析:
看了半天的surface post后消息的传递，重重的继承，各种set，（╯－＿－）╯╧╧
先写个大概流程，免得待会忘了
首先是consumer的创建 在Layer::onFirstRef()
```cpp
sp<IGraphicBufferConsumer> consumer;
BufferQueue::createBufferQueue(&producer, &consumer);
```

listener的创建 没错，mSurfaceFlingerConsumer就是个listener
```cpp
mSurfaceFlingerConsumer = new SurfaceFlingerConsumer(consumer, mTextureName);
```

SuffaceFlingerComsumer的构造函数 call call call到了… ConsumerBase::ConsumerBase proxy是ConsumerBase自身的代理
```cpp
status_t err = mConsumer->consumerConnect(proxy, controlledByApp);
```

Bp端的调用，Bn端的实现 在BufferQueueConsumer::consumerConnect,在这里做了个转发，而且还在h文件里，搞得cpp文件都没有consumerConnect，我&\*&^%!(@
```cpp
virtual status_t consumerConnect(const sp<IConsumerListener>& consumer,bool controlledByApp) {
    return connect(consumer, controlledByApp);
}
```

connect的实现
```cpp
mCore->mConsumerListener = consumerListener;
```

也就是说
```cpp
consumerListener == ConsumerBase == SuffaceFlingerComsumer
```

而下面又有
```cpp
mSurfaceFlingerConsumer->setContentsChangedListener(this);
```

相当于
```cpp
mCore->mConsumerListener == layer
```

所以queue的时候
```cpp
listener.xxx == layer.onXXX;
```

从这以后，渲染工作就交给了SurfaceFlinger服务了
