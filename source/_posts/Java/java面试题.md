---
title: java面试题
date: 2019-04-11 14:51:14
categories: Java
tags: 面试
---
准备个底在去受虐

<!-- more -->

Java 基础
---

### JDK和JRE有什么区别？
jdk是开发包，jre是运行环境
JDK是Java的开发工具，它不仅提供了Java程序运行所需的JRE，还提供了一系列的编译，运行等工具，如javac，java，javaw等。
JRE只是Java程序的运行环境，它最核心的内容就是JVM（Java虚拟机）及核心类库。


### ==和equals的区别是什么？
==比较的是对象，equals比较多是对象的内容
从原理来讲，Object的equals是由 == 实现的，讨论的比较多的String是重载了Object的equals改变了比较的方式

### 两个对象的hashCode()相同，则equal()也一定为true，对吗？
equal相等的 hashcode一定相等 反过来hashcode不相等的equal一定不相等
hashcode相等的不一定equal相等
(md5)

### final在Java中有什么作用？
对象修饰，一旦赋值不可修改
类修饰，不可继承
方法修饰，不可重载

### Java中的Math.round(-1.5)等于多少？
a + 1/2 向下取整
-1.5 + 0.5 = -1.0
a + 1/2是用位操作完成的

### String属于基础的数据类型吗？
不是，final类
stort long int double float byte char boolean 

### Java中操作字符串都有哪些类？他们之间有什么区别？
正则表达式 

### String str="i" 与 String str = new String("i")一样吗？
否，"i"在常量池，new String("i")在堆上

### 如何将字符串反转？
StringBuffer reverse  中间对调 >>

### String类的常用方法都有哪些？
length()
charAt()
substring()
replace()
indexOf()
trim()
split()

### 抽象类必须要有抽象方法吗？
可以没有抽象方法

### 普通类和抽象类有哪些区别？
抽象类不能直接使用，只能通过子类实现其所有抽象方法后才能使用

### 抽象类能使用final修饰吗？
不能 内部可以为private

### 接口和抽象类有什么区别？
方法均为public abstract
成员均为public static final
接口不可有构造方法

抽象类可以有实现方法，接口的不能有实现
类只能继承一个父类，但可以实现多个接口

### Java中IO流分为几种
流向： input output
类型：字节流 字符流
节点流 处理流 缓冲流

### BIO、NIO、AIO有什么区别
Block IO、Non-Blocking IO、Ansyc IO

### Files的常用方法都有哪些
new File 
mkdir
mkdirs
renameTo

delete
exists
isFile
isDirectory

getName
getAbsolutePath
length

list
listFiles



容器
---
### java容器都有哪些？
Collection
*   List  
*   *   ArrayList (数组)
*   *   LinkedList(双向链表)
*   Set 
*   *   HashSet(HashMap实现)
*   *   TreeSet(TreeMap实现)
*   *   *   LinkedHashSet(LinkedHashMap实现)
*   Queue
*   *   PriorityQueue

Map
*   HashMap(单链表数组)
*   *   LinkedHashMap(有序Map HashMap + 双向链表)
*   TreeMap

Iterator
*   ListIterator

Comparable & Comparator

Utilities
*   Collections
*   Arrays

### Collection和Collections有什么区别
Collections为工具类
Collection为集合接口

### List、Set、Map之间的区别是什么
列表有序，Set为Value唯一，Map为key索引

### HashMap和Hashtable有什么区别
线程安全 null-key & null-value
迭代器

### 如何决定使用HashMap还是TreeMap
HashMap 查询
TreeMap 增加、快速创建

### 说下HashMap的实现原理
数组加列表，拉链法，加载因子、扩容

### 说下HashSet的实现原理
使用HashMap实现

### ArrayList和LinkedList的区别？
ArrayList使用数组实现，查询O(1),增加O(1)，删除O(n)
LinkedList使用链表实现，查询O(n)，增加O(1)，删除O(1）

### 如何实现数组和List之间的转换？
toArray()
new Array() == Arrays.asList();
new List() + addlist


### ArrrayList和Vector的区别？
Vector是线程安全

### Queue中poll() 和remove()有什么区别
当Queue为空的处理方式不一样 

### 哪些集合类是线程安全的
HashTable Vector 

### 迭代器Iterator是什么？
一个接口 hasNext next remove

### Iterator怎么使用，有什么特点
统一了遍历方式

### Iterator和ListIterator有什么区别
索引，add，逆向遍历，set

### 怎么确保一个集合不能被修改 
Collections.unmodifiableMap

多线程
---
### 并行和并发有什么区别？
并发cpu切换
并行cpu运行不同任务

### 线程和进程的区别？
进程——资源分配的最小单位，线程——程序执行的最小单位。

内容空间:共享相同数据段，有相同的地址空间
通信方式:管道，信号，消息队列，共享内存，套接字
共享资源:文件描述符等

优缺点：线程轻量
        进程稳定安全

### 守护线程是什么
GC线程，当所有的User Thread都退出时，守护线程会自动退出

### 说一下runnable和callable有什么区别
Runnable没有返回值；Callable可以返回执行结果
Callable接口的call()方法允许抛出异常；Runnable的run()方法异常只能在内部消化，不能往上继续抛

### 线程有哪些状态
1.新建状态
2.就绪状态
等待CPU时间
3.运行状态
4.阻塞状态
sleep - IO - 获取锁 - wait()
5.死亡状态
正常运行，抛出异常 isAlive()

### sleep()和wait()有什么区别？

*   sleep()是Thread wait()是Object
*   wait阻塞当前线程，会释放synchronized的锁，

### notify()和notifyAll()有什么区别?
*   notify会释放一个wait的锁
*   notifyAll会释放所有wait的锁，并开始竞争

### 创建线程池有哪几种方式？
newSingleThreadExecutor
newFixedThreadPool
newCachedThreadPool
newScheduledThreadPool

### 线程池都有哪些状态?
running -> shutdown -> stop -> tidying -> terminated

### 线程池中submit() 和execute()方法有什么区别?
submit 有返回值，便于处理异常

### 在Java程序中怎么保证多线程的运行安全?
*   原子性 atomic,synchronized
*   可见性 synchronized,volatile
*   有序性 synchronized,LOCK

### 多线程锁定升级原理是什么？
偏向锁 没有竞争线程
轻量级锁 有一个竞争线程
重量级锁 有两个以上竞争线程

### 什么是死锁？
互斥条件
请求和保持条件
不剥夺条件
循环等待条件

### 怎么防止死锁？
加锁时许
超时
死锁检测

### ThreadLocal是什么？有哪些使用场景？

### 说一下Synchronized底层实现原理？
监视器锁计数 进入+1，释放-1
ACC_SYNCHRONIZED

### Synchronized 和 volatile的区别是什么？
*   volatile关键字解决的是内存可见性的问题  synchronized关键字解决的是执行控制的问题
*   volatile本质是在告诉jvm当前变量在寄存器（工作内存）中的值是不确定的，需要从主存中读取； synchronized则是锁定当前变量，只有当前线程可以访问该变量，其他线程被阻塞住。
*   volatile仅能使用在变量级别；synchronized则可以使用在变量、方法、和类级别的
*   volatile仅能实现变量的修改可见性，不能保证原子性；而synchronized则可以保证变量的修改可见性和原子性
*   volatile不会造成线程的阻塞；synchronized可能会造成线程的阻塞。
*   volatile标记的变量不会被编译器优化；synchronized标记的变量可以被编译器优化

### Synchronized 和 Lock有什么区别？
*   synchronized是java内置关键字，在jvm层面，Lock是个java类
*   synchronized无法判断是否获取锁的状态，Lock可以判断是否获取到锁
*   synchronized会自动释放锁(a 线程执行完同步代码会释放锁 ；b 线程执行过程中发生异常会释放锁)，Lock需在finally中手工释放锁（unlock()方法释放锁），否则容易造成线程死锁
*   用synchronized关键字的两个线程1和线程2，如果当前线程1获得锁，线程2线程等待。如果线程1阻塞，线程2则会一直等待下去，而Lock锁就不一定会等待下去，如果尝试获取不到锁，线程可以不用一直等待就结束了
*   synchronized的锁可重入、不可中断、非公平，而Lock锁可重入、可判断、可公平（两者皆可）
*   Lock锁适合大量同步的代码的同步问题，synchronized锁适合代码少量的同步问题。

### Synchronized 和 ReentrantLock区别是什么？

### 说一下Atomic的原理
volatile保证可见性
自旋 + CAS（乐观锁）保证原子性

反射
---
### 什么是反射？
反射机制指的是程序在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法。

### 什么是Java序列化？什么情况下需要序列化？
Java Serialization、Json、Hession、Dubbo、FST、Kryo
将对象的内容进行流化

### 动态代理是什么？有哪些应用？
动态代理是一种在运行时动态地创建代理对象，动态地处理代理方法调用的机制
日志
处理注释
性能

### 怎么实现动态代理？
InvocationHandler
newProxyInstance
method.invoke(hello, args);

对象拷贝
---
### 为什么要使用克隆？
想对一个对象进行处理，又想保留原有的数据进行接下来的操作

### 如何实现对象克隆？
=
clone
Cloneable
序列化

### 深拷贝和浅拷贝区别是什么？
clong应用及clone值

异常
---
### throw和throws的区别？
throws声明会抛出的异常，throw抛出具体异常

### final和finally、finalize有什么区别？
finalize GC相关

### try-catch-finall中哪个部分可以省略？
catch 或 finally 可以省略一个

### try-catch-finally中，如果catch中return了，final还会执行吗？
final会执行，return为final中的为准

### 常见的异常类有哪些？
IOException  RunntimeException

NullPointerException 、 IllegalArgumentException、ClassNotFoundException、ArithmeticException、ArrayIndexOUtOfBoundsException、InputMisMatchException、NumberFormatException

JVM
---
### 说一下JVM的主要组成部分？及其作用？ 
类加载子系统
方法区 java堆 直接内存
垃圾回收系统
java栈 每个线程均有一个栈
本地方法栈
PC寄存器 指向当前命令
执行引擎
https://www.cnblogs.com/zwbg/p/6194470.html

### 说一下JVM运行时数据区？
程序计数器  虚拟机栈 本地方法栈 堆 方法区（常量，静态变量）

### 说一下堆栈的区别？
堆是一种排过序的树形数据结构
栈是一种后进先出性质的数据结构

### 队列和栈是什么？有什么区别？

### 什么是双亲委派模型？

### 说一下类加载的执行过程？

### 怎么判断对象是否可以被回收？

### java 中都有哪些引用类型？

### 说一下JVM有哪些垃圾回收算法？

### 说一下JVM有哪些垃圾回收器？

### 详细介绍一下CMS垃圾回收器？

### 新生代垃圾回收器和老生带垃圾回收器都有哪些？

### 简述垃圾回收器是怎么工作的？

### 说一下JVM调优的工具？

### 常用的JVM调优的参数都有哪些？


































