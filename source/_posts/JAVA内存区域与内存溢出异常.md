---
title: JAVA内存区域与内存溢出异常
date: 2017-03-07
tags: Java
---
--------

# 一、概论
java不需要程序员针对new操作去写free/delete代码，这靠的就是虚拟机的自动内存管理机制，但是一旦出现内存泄露或者溢出的情况，如果不了解虚拟机是怎么使用内存的，排查错误将成为一种异常艰难的工作。

# 二、运行时数据区域
Java虚拟机在执行Java程序时会把他所管理的内存划分为几个不同的数据区域。

## 1.程序计数器
它是一块较小的内存空间，它的作用可以看成当前线程所执行字节码的行号指示器，在虚拟机的概念模型里，字节码解释器工作时就是通过改变这个计数器的值去选取下一条需要执行的字节码指令。
<!--more-->
每个线程都有一个独立的程序计数器，每条线程之间的计数器互不影响，独立存储。
如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址，如果正在执行的是Native方法,这个计数器的值则为空，此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError的区域。

## 2.Java虚拟机栈
同样也是线程私有的，它的生命周期与线程相同，它描述的是Java方法执行的内存模型：每个方法执行的时候都会同时创建一个栈帧用于存储局部变量表、操作栈、动态链接、方法出口等信息。每一个方法被调用直至执行完成都对应着一个栈帧从虚拟机中从入栈到出栈的过程。
局部变量表中存放着编译器可知的各种基本数据类型、对象引用（reference类型）和returnAddress类型。
其中64位长度的long和double类型的数据会占用2个局部变量空间，其余的数据类型只占用1个。局部变量表所需的内存空间在编译期间就完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，这个方法运行期间不会改变局部变量表的大小。
在Java虚拟机中，这个区域规定了两种异常情况：如果请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError;如果虚拟机栈能够动态扩展，当扩展时无法申请到足够的内存时将抛出OutOfMemoryError。

## 3.本地方法栈
与虚拟机栈发挥的作用是非常相似的，区别是本地方法栈是为虚拟机使用到的Native方法服务，异常抛出同上。

## 4.Java堆
对于大多数应用来说，Java堆是Java虚拟机所管理的内存中最大的一块，Java堆是所有线程共享的一块内存区域，在虚拟机启动的时候创建。该内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。这一点在Java虚拟机规范中的描述为：所有的对象实例以及数组都要在堆上分配，但是随着JIT编译器的发展与逃逸分析技术的逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化发生。
Java堆是垃圾收集器管理的主要区域。如果从内存回收的角度看，Java堆还可以分为新生代和老年代.如果从内存分配的角度看，线程共享的Java堆还可以分为多个线程私有的分配缓存区。不过不管怎么划分，都与存放内容无关。
Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的就行。在实现时，可以实现成固定大小的，也可以是可扩展的，不过一般的都是安装可扩展来实现的（通过-Xmx和-Xms控制）。如果堆中没有内存完成实例分配，并且堆也无法得到再扩展时，就会抛出OutOfMemoryError。

## 5.方法区
与堆一样都是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。
在HotSpot虚拟机中它被称作“永久代”，因为HotSpot把GC分代收集扩展到了方法区。其他的虚拟机没有永久代的概念。
它和Java堆一样不需要连续的存放空间和可以选择固定大小或者可扩展外，还可以选择不实现垃圾收集，一样在无法满足内存分配需求时抛出OutOfMemoryError。

## 6.运行时常量池
JDK1.6之前是方法区的一部分，JDK1.7被移到堆中管理，具有动态性，用于存放编译器生成的各种字面量和符号引用。
运行时常量池相对与Class文件常量池的一个重要特征就是具有动态性，Java语言并不要求常量一定要在编译器产生，运行期间也可以将新的常量放入池中，这个特性被开发人员用的比较多的就是String的intern()方法。
同样，可能抛出OutOfMemoryError。


# 三、对象访问
Object obj = new Object（）；
在这句代码中Object obj将会反应到Java的本地变量表，作为一个reference类型数据出现，而new Object()将会反应到Java堆中，形成一块存储Object类型所有实例数据值的结构化内存。另外，在Java堆中还必须包含能查找到此对象类型数据（如父类，对象类型，接口，方法等）的地址信息，这些类型数据将存储在方法区中。
由于reference中只规定了一个指向对象的引用，并没有定义这个引用该如何去定位，主流的定位方法有两种：
- 使用句柄访问方式，Java堆中划分一块内存作为句柄池，reference中存储的是对象的句柄对象，而句柄中包含了对象实例数据和类型信各自的具体地址信息。较稳定，对象移动只需改变句柄中的实例数据指针。
- 直接指针访问，reference存储的就是对象地址。速度更快。

# 四、OutOfMemoryError异常

## 1.Java堆溢出
在Java Debug Configuration中可以设置参数
-verbose:gc -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
再运行以下程序：
```java
public class HeapOOM {
public static void main(String[] args){
	List<OOMObject>  list=new ArrayList<>();
			while(true){
				list.add(new OOMObject());
			}                 
}
static class OOMObject{
}
}
```
运行结果
Exception in thread “main” java.lang.OutOfMemoryError: Java heap space
at java.util.Arrays.copyOf(Arrays.java:3210)
at java.util.Arrays.copyOf(Arrays.java:3181)
at java.util.ArrayList.grow(ArrayList.java:261)
at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:235)
at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:227)
at java.util.ArrayList.add(ArrayList.java:458)
at HeapOOM.main(HeapOOM.java:8)

## 2.Java虚拟机栈和本地方法栈溢出
在HotSpot虚拟机中不区分虚拟机栈和本地方法栈，因此对于Hotspot虚拟机，-Xoss参数（设置本地方法栈的大小）实际上是无效的，栈容量只由-Xss参数决定。
-Xss228K
```java
public class StackSOF {
	private int stackLength=1;
	public void stackLeak(){
		stackLength++;
		stackLeak();
	}
	public static void main(String[] args){
		StackSOF  sof=new StackSOF();
		try{
		sof.stackLeak();
		}catch(Throwable e){
			System.out.println("stack length:"+sof.stackLength);
			throw e;
		}
	}
}

```
运行结果：
Exception in thread “main” java.lang.StackOverflowError
at StackSOF.stackLeak(StackSOF.java:7)
at StackSOF.stackLeak(StackSOF.java:7)
at StackSOF.stackLeak(StackSOF.java:7)
at StackSOF.stackLeak(StackSOF.java:7)…

## 3.运行时常量池溢出
向运行池常量池中添加内容，最简单的做法是使用String.intern()这个Native方法，它的作用是：如果池中包含了一个等于此String对象的字符串，那么返回代表池中这个字符串的String对象；否则，将此String对象包含的字符串添加到常量池中，并返回此String对象的引用。由于常量池分配在方法区内，可以通过-XX:PermSize和-XX:MaxPermSize（在HotSpot 8.0中忽略了这两个参数）限制方法区的大小。





