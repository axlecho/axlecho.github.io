---
title: java
date: 2019-01-22 12:09:59
tags: android
---

######虚拟机
对象创建 new

常量池
解析class（加载，解析，初始化 classloader）
分配内存
内存布局

对象头
实例数据
对齐
访问

句柄池 （类型指针，实例指针）
内存区域

方法区 （类信息，常量，静态常量）
堆（实例对象）
虚拟机栈（局部变量表，操作数栈，动态连接，方法出口）
本地方法栈（Native方法）
线程计数器（线程当前执行的字节码行号指示器）
Java内存模型

目的：兼容各种平台
目标：java的所有变量都存储在主内存中，线程拥有共享变量副本
8种操作：lock，unlock，read，load，use，assign，store，write
lock – unlock次数相同才能平衡
加载器

生命周期 – 加载，验证，准备，解析，初始化，使用和卸载
触发时机
new，读取设置一个类的静态字段，调用一个类的静态方法
反射调用
父类
入口类
java.lang.invoke.MethodHandle
加载流程

加载
全限定名获取二进制字节流
静态结构转换为运行时数据结构
生成内存代表类的java.lang.Class
验证
文件格式(class文件格式，版本）
元数据（语义）
字节码验证（数据流和控制流）
符号引用验证
准备
初始化变量
解析
将常量池符号替换为直接引用
初始化
对类变量进行赋值及执行静态代码块
类加载器
负责加载流程

启动类加载器（Bootstrap ClassLoader） 核心类
扩展类记载器（Extension ClassLoader） 扩展的核心类
应用程序类加载器（Application ClassLoader）用户classpath上指定的类 （可直接ClassLoader.getSystemClassLoader()获取）
双亲委派模型，优先级
GC算法

标记-清除
复制算法
标记-整理
分代搜集 Enden - Survivor ：Survivor - 年老代 - 永久代
Minor GC && Full GC/Major GC
GC搜集器

Serial收集器
ParNew收集器
Parallel Scavenge收集器
Serial Old收集器
Parallel Old收集器
CMS（Concurrent Mark Sweep）收集器 并发
G1收集器 并发 分代
判断可回收算法

引用计数
可达性分析算法
计算gc roots（栈帧的本地变量表，静态属性引用，常量引用，jni引用）
F-Queue
引用类型

强引用 有用，内存异常都不回收
软引用 有用但非必须 内存即将异常时回收
弱引用 不会阻止垃圾回收
虚引用 对象被收集器回收收到一个系统通知
Activity
Activity的生命周期

onCreate -> onStart -> onResume -> onPause -> onStop -> onDestroy
↑ |
| ↓
———— onRestart ———–

第一次启动 onCreate -> onStart -> onResume
打开新Activity onPause -> onStop
回到原来的Activity onRestart -> onStart -> onResume
按back返回 onPause -> onStop -> onDestory
按home键 onPause -> onStop -> onRestart -> onStart -> onResume
finish onDestory

横竖屏切换
onStop之前 onSaveInstanceState && onStart之后 onRestoreInstanceState
onPause -> onSaveInstanceState -> onStop -> onDestory -> onCreate -> onStart -> onRestoreInstanceState -> onResume

android:configChanges = “orientation| screenSize” 避免横竖屏切换销毁activity

1
2
3
4
@Override
    public void onConfigurationChanged(Configuration newConfig) {
        super.onConfigurationChanged(newConfig);
    }
activity优先级
前台activity Resumed（活动状态）
可见activity Paused（暂停状态）
后台activity Stopped（停止状态）

启动模式
标准模式（standard）
栈顶复用模式（singleTop）
栈内复用模式（singleTask）
单例模式（singleInstance）

singleTop 防重复点击，通知栏

1
2
3
4
@Override
protected void onNewIntent(Intent intent) {
    super.onNewIntent(intent);
}
singleTask 单例模式的activity 主界面
singleInstance 单例模式的activity加强版 桌面启动，呼叫来电

特殊情况：调用SingleTask模式的后台任务栈中的Activity，会把整个栈的Actvity压入当前栈的栈顶。singleTask会具有clearTop特性，把之上的栈内Activity清除。

Activity flags
FLAG_ACTIVITY_NEW_TASK
FLAG_ACTIVITY_SINGLE_TOP
FLAG_ACTIVITY_CLEAR_TOP

BroadcastReceiver
模型：观察者模式

订阅者
发布者
消息中心 AM
注册：

静态注册 在AndroidMainifest.mxl首次启动app，会实例化该广播并注册到系统
动态注册 使用Context的registerReceiver
动态广播 注册必须注销，不然会有内存泄漏
广播种类

普通广播
系统广播
有序广播
应用内广播(exported = false，permission，setPackage)
ContentProvider
数据交互与共享

ContentProvider的使用

统一资源识别符 schema:authorith:path:id
mime /
表数据
增、删、查、改
ContentResolver - cursor

ContentUris
UriMatcher
ContentObserver

进程内通信
进程间通信

1
2
3
<provider android:name="MyProvider"
    android:authorities="cn.scu.myprovider"
/>
优点:安全、访问简单&高效，(数据源的adapter)

消息机制
Message – MessageQueue – Handler –Looper

handler.sendMessage –> MessageQueue -> Looper.loop(Message.next) –> handler.dispatchMessage -> handler.handleMessage

流程

looper – Looper.prepare – Looper.loop
handler – (post && send)sendMessageAtTime – MessageQueue.enqueueMessage – MessageQueue.next – handler.dispatchMessage
事件分发机制
分发MotionEvent
流程

驱动捕获 -> Activity（Window） -> ViewGroup -> View
dispatchTouchEvent() – onInterceptTouchEvent – onTouchEvent() return true(消费该事件) false (拦截消息) super
默认情况

dispatchTouchEvent:activity -> viewgroup -> view
onTouchEvent:view -> viewgroup -> activity
viewgroup重载onInterceptTouchEvent() 并返回true，事件不再传递到view
调用onTouch,activity不再处理事件

View消费事件的条件

enable
onTouch 返回true
onTouch 高于onClick
事件的连锁机制
ACTION_DWON return true后，ACTION_UP与ACTION_MOVE才能收到
onInterceptTouchEvent

异步
AsyncTask其实是个线程池

实现:SerialExecutor，THREAD_POOL_EXECUTOR，InternalHandler

核心方法

onPreExecute() // 主线程
doInBackgroud() // 子线程
onProgressUpdate(Progress…) // 主线程
onPostExecute(Result) // 主线程
onPreExecute() –> doInBackground() –> publishProgress() –> onProgressUpdate() –> onPostExecute()
cancel 设置cancel标记位
note:InternalHandler是一个静态类，为了能够将执行环境切换到主线程，因此这个类必须在主线程中进行加载。所以变相要求AsyncTask的类必须在主线程中进行加载。

note:使用不当会引起不良后果

生命周期 在onDestroy内调用cancel
内存泄漏
结果丢失
