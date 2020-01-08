---
title: Android总结
date: 2019-07-09 16:57:30
categories: Android
tags:
	- android
	- 技术总结
---

Android的一些基础总结
<!-- more -->

Activity
---

### Activity的生命周期

onCreate -> onStart -> onResume -> onPause -> onStop -> onDestroy
↑ |
| ↓
onRestart

**各种场景流程**
第一次启动 onCreate -> onStart -> onResume
打开新Activity onPause(A) -> onCreate(B) -> onStart(B) -> onResume(B) -> onStop(A)
按back返回原来的Activity onPause(B) onRestart(A)-> -> onStart(A) -> onResume(A) -> onStop(B) -> onDestory(B)
按home键 onPause -> onStop -> onRestart -> onStart -> onResume
finish onDestory

**横竖屏切换**
onStop之前 onSaveInstanceState && onStart之后 onRestoreInstanceState
onPause -> onSaveInstanceState -> onStop -> onDestory -> onCreate -> onStart -> onRestoreInstanceState -> onResume
避免横竖屏切换销毁activity
```java
android:configChanges = "orientation\| screenSize" 

@Override
public void onConfigurationChanged(Configuration newConfig) {
    super.onConfigurationChanged(newConfig);
}
```

**activity优先级**
前台activity Resumed（活动状态）
可见activity Paused（暂停状态）
后台activity Stopped（停止状态）

**启动模式**
标准模式（standard）
栈顶复用模式（singleTop）
栈内复用模式（singleTask）
单例模式（singleInstance）

singleTop 防重复点击，通知栏

```java
@Override
protected void onNewIntent(Intent intent) {
    super.onNewIntent(intent);
}
```

singleTask 单例模式的activity 主界面
singleInstance 单例模式的activity加强版 桌面启动，呼叫来电

特殊情况：调用SingleTask模式的后台任务栈中的Activity，会把整个栈的Actvity压入当前栈的栈顶。singleTask会具有clearTop特性，把之上的栈内Activity清除。

Activity flags
FLAG_ACTIVITY_NEW_TASK
FLAG_ACTIVITY_SINGLE_TOP
FLAG_ACTIVITY_CLEAR_TOP

BroadcastReceiver
---

模型：观察者模式
订阅者 - 发布者 - 消息中心 AM

静态注册 在AndroidMainifest.mxl首次启动app，会实例化该广播并注册到系统
动态注册 使用Context的registerReceiver
动态广播 注册必须注销，不然会有内存泄漏

### 广播种类
普通广播
系统广播
有序广播
应用内广播(exported = false，permission，setPackage)

ContentProvider
---
数据交互与共享

**ContentProvider的使用**

统一资源识别符 schema:authorith:path:id
mime /
表数据
增、删、查、改
ContentResolver - cursor
ContentUris
UriMatcher
ContentObserver

```xml
<provider android:name="MyProvider"
    android:authorities="cn.scu.myprovider"
/>
```

优点:安全、访问简单&高效，(数据源的adapter)

消息机制
---
**流程**
Message – MessageQueue – Handler –Looper
handler.sendMessage –> MessageQueue -> Looper.loop(Message.next) –> handler.dispatchMessage -> handler.handleMessage
looper – Looper.prepare – Looper.loop
handler – (post && send)sendMessageAtTime – MessageQueue.enqueueMessage – MessageQueue.next – handler.dispatchMessage

事件分发机制
---
分发MotionEvent

**流程**
驱动捕获 -> Activity（Window） -> ViewGroup -> View
dispatchTouchEvent() – onInterceptTouchEvent – onTouchEvent() return true(消费该事件) false (拦截消息) super

**默认情况**
dispatchTouchEvent:activity -> viewgroup -> view
onTouchEvent:view -> viewgroup -> activity
viewgroup重载onInterceptTouchEvent() 并返回true，事件不再传递到view
调用onTouch,activity不再处理事件

**View消费事件的条件**
enable
onTouch 返回true
onTouch 高于onClick

**事件的连锁机制**
ACTION_DWON return true后，ACTION_UP与ACTION_MOVE才能收到
onInterceptTouchEvent

异步
---
AsyncTask其实是个线程池

实现:SerialExecutor，THREAD_POOL_EXECUTOR，InternalHandler

**核心方法**
onPreExecute() // 主线程
doInBackgroud() // 子线程
onProgressUpdate(Progress…) // 主线程
onPostExecute(Result) // 主线程
onPreExecute() –> doInBackground() –> publishProgress() –> onProgressUpdate() –> onPostExecute()
cancel 设置cancel标记位
note:InternalHandler是一个静态类，为了能够将执行环境切换到主线程，因此这个类必须在主线程中进行加载。所以变相要求AsyncTask的类必须在主线程中进行加载。

note:使用不当会引起不良后果
例如在生命周期onDestroy内调用cancel 会造成内存泄漏、结果丢失


跨进程
---
**Android跨进程形式**
Activity隐式启动
Content provider
Broadcast
Remote Service(startService、AIDL）

Binder有什么优点
高效、安全（Uid鉴权）
