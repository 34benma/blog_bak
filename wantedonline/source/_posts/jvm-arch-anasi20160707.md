---
title: Java虚拟机架构分析
date: 2016-07-07 19:11:34
tags: 
  - JVM
  - 架构分析
  - 内存溢出
categories: 深入理解
---
Java虚拟机(JVM)是装载执行Class文件的唯一场所。Java之所以能够实现跨平台运行就是因为JVM屏蔽了不同操作系统的底层实现，统一了一套Java指令。实现了在不同操作系统的JVM都可以执行同一套Java指令。JVM事实上充当了代理的角色，将不同操作系统的底层不同屏蔽起来，对上层提供统一的接口。

<!--more-->

JVM的体系结构如下图所示：
![JVM体系架构](http://o9z6i1a1s.bkt.clouddn.com/1.png)

<h2>JVM的体系结构</h2>

<h3>运行时数据区</h3>

运行时数据区是JVM运行时的数据所在地。这些数据包括类数据，线程上下文数据，程序执行的数据等。因此可以把运行时数据区理解为JVM的数据仓库。程序运行的所有数据都来自这里。运行时数据区包括方法区、堆、Java虚拟机栈、本地方法栈、程序计数器。

<h3>程序计数器(线程私有)</h3>

当前线程所执行的的字节码行号指示器。虚拟机通过改变这个计数器的值来选取下一条需要执行的指令，每条线程都有一个独立的计数器，因为线程的切换需要记住上一次线程执行的位置。但如果执行的是native方法，则计数器值为空(Undefined)。

<h3>虚拟机栈(线程私有)</h3>

Java虚拟机栈是线程私有的，其生命周期和线程一样。虚拟机栈里保存的是当前线程所执行到的方法的上下文信息。每调用一个方法，就会有一次入栈。入栈的对象叫做栈帧(Stack Frame),这些方法的上下文信息包括局部变量表，操作数栈，动态链接，方法出口等。当方法执行完后，对应一次出栈。

<h3>本地方法栈</h3>

类似于虚拟机栈，区别在于Java虚拟机栈执行的是虚拟机方法，本地方法栈执行的是本地Native方法。

<h3>堆(线程共享)</h3>

Java堆是Java虚拟机所管理的内存中最大的一块。用于存放对象的实例，几乎所有的对象实例都在这里分配（并不是绝对）。

<h3>方法区(线程共享)</h3>

用于存储虚拟机加载的类信息，常量，静态变量等数据。Class文件中有一项叫做常量池(Constant Pool Table)，用于存放编译期生成的各种字面量和符号引用，这部分内容在类加载后就是放在方法区的运行时常量池中。事实上，在虚拟机运行时也可以将字符串常量放到运行时常量池中。JDK1.7之前的String.intern()方法就可以实现。

<h3>直接内存</h3>

在JDK1.4中加入的NIO有一个缓冲类叫ByteBuf，这个类有一个DirectByteBuffer方法，调用该方法得到的内存缓存就是直接内存。这块内存不再是Java虚拟机管理，而是操作系统直接管理。这样做的好处时再一些IO场景中可以显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。

<h3>执行引擎</h3>

执行引擎是JVM执行具体的Java指令的场所。它从运行时数据区中获取数据执行(输入)，执行完将结果再次返回运行时数据区。

<h3>本地库接口</h3>

Java可以通过JNI调用本地方法。这些接口就是本地库接口。调用的那些方法库叫本地方法库。通常是用C/C++写的一些本地函数库。这样做的好处是可以提高性能。

<h2>JVM常见内存溢出分析</h2>

所谓内存溢出就是程序运行时需要申请的内存大于目前虚拟机所在区域可用的内存。这时候程序无法运行下去，虚拟机抛出内存溢出异常，程序退出执行。能够发生内存溢出的区域有方法区，Java堆，虚拟机栈，本地方法栈。理论上直接内存也有可能发生内存溢出。程序计数器因为保存的是当前线程下一条执行指令的地址，因此不会发生内存溢出。

<h3>方法区内存溢出</h3>

方法区在HotSpot也可以称为永久代（Permanent Generation）。可以通过-XX:PermSize和-XX:MaxPermSize两个参数调节其初始大小和最大值。为了构造方法区内存溢出，需要将这两个值调整一下。这两个值默认分别是物理内存的1/64和1/4。

设置启动参数为
```
    -XX:PermSize=1M -XX:MaxPermSize=2M
```
写一个只含有main函数的方法，里面什么也不干。运行程序发现马上会抛出以下异常：

![方法区内存溢出异常信息](http://o9z6i1a1s.bkt.clouddn.com/methodarea.png)

从图中可以看出，当虚拟机启动加载类的常量池的时候发现内存不够，因此启动中断，抛出PermGen space OOM。

如果没有特殊需求，一般不需要设置这两个值。

<h3>堆内存溢出</h3>

Java堆是用于存放Java运行时各种对象等数据实例。因此当新产生的对象实例所需要的内存大于目前Java堆空余空间大小而且Java堆中对象实例都不能被垃圾回收器所回收也不能升级为老年代或直接在老年代分配的时候就会报Java堆OOM异常。

从上面的描述来看，堆内存溢出产生的条件还是比较复杂。事实上，HotSpot虚拟机的Java堆进一步还可以分为新生代，老年代。新生代里面有Eden区，From Survivor空间，To Survivor空间等概念。这个涉及到JVM的垃圾回收机制。将在下一篇详述。

因为Java堆内存的复杂以及垃圾回收算法影响。要分析清楚堆内存溢出的原因还是有点困难。目前，可以理解为new出来的对象需要的内存大于目前堆中可用的内存。

想要构造一个简单的Java堆内存溢出，可以如下操作
设置启动参数:
```
-Xms20M -Xmx20M
```
    
    /**
     * @author wangcheng
     * @Date 2016年7月7日 下午5:31:08
     * @Desc:
     * -Xms20M -Xmx20M
     */
    public class HeapOOM {
	
	    public static void main(String[] args) {
		    byte[] b = new byte[1024*1024*100];
	    }
    }


将初始堆大小和最大堆大小都设置成20M，在程序运行的时候，在堆中new一个100M大小的byte数组。因为最大堆大小都没有100M，这时候将会抛出Java heap space OOM异常。如图：

![堆内存溢出异常信息](http://o9z6i1a1s.bkt.clouddn.com/heap.png)

上文说过，Java堆分为新生代，老年代等，还和垃圾回收以及引用强弱等因素，堆内存溢出的场景远比这个复杂得多。

<h3>虚拟机栈栈溢出</h3>

虚拟机栈是线程私有，这是一个栈结构，因此有两种类型的内存溢出。一种是栈的深度超过虚拟机所允许的最大深度抛出StackOverflowError，一种是当前的栈帧所需要的内存超过限制抛出OutOfMemoryError。

本地方法栈和虚拟机栈类似，除了执行的是本地方法。要构造出本地方法栈溢出还是有些难度。需要很深的native调用。

下面构造一种简单的虚拟机栈栈溢出。
设置启动参数：
```
-Xss:1K
```

    /**
 	 * @author wangcheng
	 * @Date 2016年7月7日 下午6:40:49
	 * @Desc:
	 * -Xss:1K -XX:ThreadStackSize=1K
	 */
	public class StackOFE {
		public void call() {
			call();
		}
	
		public static void main(String[] args) {
			StackOFE om = new StackOFE();
			om.call();
		}

	}


这里将Java虚拟机栈大小设置为1K，然后不断地递归调用方法，每调用一次方法将会入栈一次。最终导致栈溢出。和 -Xss类似的参数还有一个叫-XX:ThreadStackSize=1K

![栈溢出异常信息](http://o9z6i1a1s.bkt.clouddn.com/stack.png)

<h3>直接内存溢出</h3>

前文说过，NIO中有一个ByteBuffer可以操作直接内存。JVM可以通过
```
-XX:MaxDirectMemorySize
```
这个参数设置直接内存大小。我们将其设置为1M。方法代码如下：

    /**
 	 * @author wangcheng
	 * @Date 2016年7月7日 下午6:54:00
	 * @Desc:
	 * -XX:MaxDirectMemorySize=1M
	 */
	public class DirectOOM {
		public static void main(String[] args) {
			ByteBuffer buffer = ByteBuffer.allocateDirect(1024*1024*10);
			buffer.clear();
		}

	}

直接分配10M的内存，将会抛出如下异常：

![直接内存溢出异常信息](http://o9z6i1a1s.bkt.clouddn.com/dm.png)

<h2>总结</h2>

本文简要分析了JVM的架构，构造了几种简单的内存溢出场景来理解JVM的内存功能块的作用。实际生产场景中，内存溢出比这个难得多。可能需要加日志手段或获取内存快照等手段长时间分析才能分析出来原因。

**注:本文只适用于HotSpot Java7**

*参考文献：*

*http://www.oracle.com/technetwork/java/tuning-139912.html*

*《深入理解Java虚拟机》 --周志明著*


*附*
[Java7虚拟机规范官方文档](http://o9z6i1a1s.bkt.clouddn.com/jvms7.pdf)
[Java8虚拟机规范官方文档](http://o9z6i1a1s.bkt.clouddn.com/jvms8.pdf)


