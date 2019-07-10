---
title: java
date: 2019-01-22 12:09:59
categories: Java
tags:
	- java
	- 技术总结
---

Java底层机制总结

<!-- more -->

JVM虚拟机
===
*A Java virtual machine (JVM) is a virtual machine that enables a computer to run Java programs as well as programs written in other languages that are also compiled to Java bytecode*

JVM主要功能模块包括类加载器、执行引擎和垃圾回收系统

内存区域
---
方法区 （类信息，常量，静态常量）
堆（实例对象）
虚拟机栈（局部变量表，操作数栈，动态连接，方法出口）
本地方法栈（Native方法）
线程计数器（线程当前执行的字节码行号指示器）

java的所有变量都存储在主内存中，线程拥有共享变量副本
内存8种操作：lock，unlock，read，load，use，assign，store，write
lock – unlock次数相同才能平衡

### 类的内存结构
对象头 [Java对象头详解](https://www.jianshu.com/p/3d38cba67f8b)
实例数据
对齐填充字节

类加载器的加载进程
---
类的生命周期 – 加载，验证，准备，解析，初始化，使用和卸载
[Java虚拟机类加载的过程](https://blog.csdn.net/u010805617/article/details/77802739)

### 触发时机
*	new，读取设置一个类的静态字段，调用一个类的静态方法
*	反射调用
*	父类
*	入口类（包涵main的类）
*	java.lang.invoke.MethodHandle

### 加载流程
加载
*	全限定名获取二进制字节流
*	静态结构转换为运行时数据结构
*	生成内存代表类的java.lang.Class

验证
*	文件格式(class文件格式，版本）
*	元数据（语义）
*	字节码验证（数据流和控制流）
*	符号引用验证

准备
*	初始化变量

解析
*	将常量池符号替换为直接引用

初始化
*	对类变量进行赋值及执行静态代码块

### 加载器种类
启动类加载器（Bootstrap ClassLoader） 核心类
扩展类记载器（Extension ClassLoader） 扩展的核心类
应用程序类加载器（Application ClassLoader）用户classpath上指定的类 （可直接ClassLoader.getSystemClassLoader()获取）

### 双亲委派模型
加载优先级 启动类加载器 > 扩展类记载器 > 应用程序类加载器


GC算法
---
[Java虚拟机详解04----GC算法和种类【重要】](https://www.cnblogs.com/smyhvae/p/4744233.html)
GC：Garbage Collection 垃圾收集
这里所谓的垃圾指的是在系统运行过程当中所产生的一些无用的对象，这些对象占据着一定的内存空间，如果长期不被释放，可能导致OOM
Java中，GC的对象是Java堆和方法区（即永久区）

### 判断回收算法
引用计数算法
根搜索算法（栈帧的本地变量表，静态属性引用，常量引用，jni引用）

### 现代的GC算法
标记-清除
复制算法
标记-整理
分代搜集 Enden - Survivor:Survivor - 年老代 - 永久代
Minor GC && Full GC/Major GC

### GC搜集器
Serial收集器
ParNew收集器
Parallel Scavenge收集器
Serial Old收集器
Parallel Old收集器
CMS（Concurrent Mark Sweep）收集器 并发
G1收集器 并发 分代

### 引用类型
强引用 有用，内存异常都不回收
软引用 有用但非必须 内存即将异常时回收
弱引用 不会阻止垃圾回收
虚引用 对象被收集器回收收到一个系统通知

finalize()可以在标记后复活对象，重载了finalize()的对象在标记后会进入一个F-Queue