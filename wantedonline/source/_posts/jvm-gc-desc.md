---
title: Java垃圾回收三大问
date: 2016-07-12 11:18:09
tags: 
 - JVM
 - 垃圾回收
categories: 深入理解
---
Java语言的一大优势在于JVM自己管理内存。解决了程序中废弃对象的内存回收问题。不再需要程序员主动去申请和释放内存，从而避免很多恼人的内存泄露问题。但是哪些内存需要回收？什么时候回收？如何回收？这是JVM垃圾回收机制需要解决的三个问题。

<!--more-->

<h2>哪些内存需要回收？</h2>

根据JVM的架构，需要进行内存回收的区域事实上只有方法区和堆区。这两个区域都是线程共享的。其他像Java虚拟机栈，本地方法栈等不需要内存回收，因为他们是线程私有，而且是栈结构。当方法或线程执行完自动会将栈帧或栈结构释放掉。程序计数器也是线程私有，而且只是保存当前线程执行下一条指令的地址。所需要的内存很小而且固定。自然也不需要专门来进行内存管理。因此JVM中只有线程共用的Java堆和方法区需要进行垃圾回收。这些垃圾主要是类信息，常量，最多的应该是对象实例。

<h2>什么时候回收？</h2>

对象回收的时机必然是该对象首先要成为一个内存垃圾。事实上Java中对象存活的判断并不是用的引用计数算法(CPython等是引用计数算法)，而是用的可达性分析算法。这个算法简单的说就是讲某些常量或类变量当成根，如果从根开始到某个对象实例有可达的路径则该对象是存活的。如果某些对象和根不再有引用关系则认为这些对象已经是垃圾了，将在垃圾算法运行的时候被回收。Java是所以采用可达性分析算法的一个原因是解决循环引用的问题。引用计数算法对于循环引用无能为力。而可达性分析算法只要对象实例和根没有引用关系则认为是垃圾。即便循环引用也没有关系。

那是不是只要对象实例对于根对象而言不可达就会被立马回收掉呢？事实上不是的。JVM专门有一个线程叫做GC线程，专门用于垃圾回收。如果是一旦发现垃圾立马回收，显然效率太低，将会严重降低JVM的吞吐量。如果采用定时扫描自然会好很多。

因此，JVM中方法区也好，堆区也好。当有类对象或对象实例成为垃圾时，如果进入了垃圾回收区并被GC扫描到就会被回收。

这个垃圾回收区在HotSpot里叫做安全点(Safepoint)，当程序执行到安全点时，将暂停下来(Stop The World)，执行GC。

<h2>如何回收？</h2>

说清楚了哪些内存需要回收，什么时候可以进行垃圾回收。就应该说一说JVM如何进行垃圾回收了。

Java7的HotSpot的垃圾回收算法不止一个，根据不同的场景(JVM运行时-client模式还是-server模式，是在新生代，老年代还是方法区)有不同的回收算法。

![JDK7 HotSpot中的GC线程](http://o9z6i1a1s.bkt.clouddn.com/gcthread.png)
![](http://o9z6i1a1s.bkt.clouddn.com/gcthread2.png)

关于JVM中采用的各种回收算法，在Oracle的这篇[博客](https://blogs.oracle.com/jonthecollector/entry/our_collectors)中写的比较官方。


**注:本文只适用于HotSpot Java7**

*参考文献*

*《深入理解Java虚拟机》 –周志明著*
*https://blogs.oracle.com/jonthecollector/entry/our_collectors*

