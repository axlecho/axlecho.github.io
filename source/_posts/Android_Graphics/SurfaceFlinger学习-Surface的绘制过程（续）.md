---
title: SurfaceFlinger学习--Surface的绘制过程（续）
date: 2019-01-12 20:30:49
categories: Android_Graphics
tags: 
    - framework
    - graphics
---

![cover](/images/ape_fwk_graphics.png)
上一篇说到surfaceflinger绘制就没了，因为surfaceflinger的流程太复杂了，有vscny信号，有messagequeue，等等，所以，主要是因为懒啦，所以先分析关于surfaceflinger的核心函数handleMessageRefresh

```cpp
void SurfaceFlinger::handleMessageRefresh() {
    ATRACE_CALL();
    preComposition();
    rebuildLayerStacks();
    setUpHWComposer();
#ifdef QCOM_BSP
    setUpTiledDr();
#endif
    doDebugFlashRegions();
    doComposition();
    postComposition();
}
```
主要工作就是合成layer与显示

一个一个来仔细看- -

---
preComposition 判断是否有未完成的绘制
```cpp
void SurfaceFlinger::preComposition()
{
    bool needExtraInvalidate = false;
    const LayerVector& layers(mDrawingState.layersSortedByZ);
    const size_t count = layers.size();
    // 调用每个layer的onPreComposition 判读是否有未完成的绘制。
    for (size_t i=0 ; i<count ; i++) {
        if (layers[i]->onPreComposition()) {
            needExtraInvalidate = true;
        }
    }

    // 没看懂啦~
    if (needExtraInvalidate) {
        signalLayerUpdate();
    }
}
```

---
rebuildLayerStacks 重新计算layer需要重绘的区域
```cpp
void SurfaceFlinger::rebuildLayerStacks() {
    // ...(奇怪的宏定义)

    // rebuild the visible layer list per screen
    // 对每个屏幕可见的layer进行rebuild
    // ...(循环每个屏幕)
    // ...(定义各种变量)
    // 计算每一个layer的可见区域
    SurfaceFlinger::computeVisibleRegions(dpyId, layers,hw->getLayerStack(), dirtyRegion, opaqueRegion);
    // ...(设置计算结果)

}
```

---
setUpHWComposer 设置HWComposer
```cpp
void SurfaceFlinger::setUpHWComposer() {
    // ...(循环所有屏幕)
    bool dirty = !mDisplays[dpy]->getDirtyRegion(false).isEmpty();
    bool empty = mDisplays[dpy]->getVisibleLayersSortedByZ().size() == 0;
    bool wasEmpty = !mDisplays[dpy]->lastCompositionHadVisibleLayers;
    bool mustRecompose = dirty && !(empty && wasEmpty);

    // new hwc
    HWComposer& hwc(getHwComposer());
    if (hwc.initCheck() == NO_ERROR) {
        // build the h/w work list
        // ...(不知道在干啥)

        // set the per-frame data
        // ...(各种种种)
        // ...(奇怪的宏定义)
        // ...(各种获取layer)
        // 做了一些变换
        layer->setPerFrameData(hw, *cur);
        // ...（奇怪的宏定义)

        // If possible, attempt to use the cursor overlay on each display.
        // ...(不知道在干啥)
    }

    status_t err = hwc.prepare();
    ALOGE_IF(err, "HWComposer::prepare failed (%s)", strerror(-err));

    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
        sp<const DisplayDevice> hw(mDisplays[dpy]);
        hw->prepareFrame(hwc);
    }
}
```

---
doDebugFlashRegions 绘制调试区域，就是开发者选项–显示surface更新那一闪一闪的东西，可以学习渲染的原理
```cpp
void SurfaceFlinger::doDebugFlashRegions()
{
    // is debugging enabled
    if (CC_LIKELY(!mDebugRegion)) return;

    const bool repaintEverything = mRepaintEverything;
    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
        const sp<DisplayDevice>& hw(mDisplays[dpy]);

        if (hw->isDisplayOn()) {
            const int32_t height = hw->getHeight();
            RenderEngine& engine(getRenderEngine());
            // ...(又是奇怪的宏，直接无视)
            {
                // transform the dirty region into this screen's coordinate space
                const Region dirtyRegion(hw->getDirtyRegion(repaintEverything));
                if (!dirtyRegion.isEmpty()) {
                   // redraw the whole screen
                   doComposeSurfaces(hw, Region(hw->bounds()));
                   // and draw the dirty region
                   // ...（继续无视)
                   // 注意这里，类似与绘图了 一个Layer有两片buffer，一片用于绘制，一片用于显示，绘制完交换
                   engine.fillRegionWithColor(dirtyRegion, height, 1, 0, 1, 1);
                   hw->compositionComplete();
                   hw->swapBuffers(getHwComposer());
               }
            }
        }
    }

    // 留意下这个，好像也很重要
    postFramebuffer();
    // ...(延时??)
    HWComposer& hwc(getHwComposer());
    if (hwc.initCheck() == NO_ERROR) {
        status_t err = hwc.prepare();
        ALOGE_IF(err, "HWComposer::prepare failed (%s)", strerror(-err));
    }
}
```

---
感觉自己越来越懒啦