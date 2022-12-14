---
title: JAVA垃圾收集
date: 2017-04-12
tags: Java
---
-------
Java虚拟机的内存区域中，程序计数器、虚拟机栈和本地方法栈三个区域是线程私有的，随线程生而生，随线程灭而灭；栈中的栈帧随着方法的进入和退出而进行入栈和出栈操作，每个栈帧中分配多少内存基本上是在类结构确定下来时就已知的，因此这三个区域的内存分配和回收都具有确定性。垃圾回收重点关注的是堆和方法区部分的内存。

Java提供finalize()方法，垃圾回收器准备释放内存的时候，会先调用finalize()。
- 对象不一定会被回收。
- 垃圾回收不是析构函数。
- 垃圾回收只与内存有关。
- 垃圾回收和finalize()都是靠不住的，只要JVM还没有快到耗尽内存的地步，它是不会浪费时间进行垃圾回收。
<!--more-->

# 引用计数算法
给对象添加一个引用计数器，一旦有一个地方引用它时，计数器加1，引用失效时，减1，任何引用计数为0的随时都是可能被回收的。
引用计数算法实现简单，判定效率也高，微软的COM技术、ActionScript、Python等都使用了引用计数算法进行内存管理,但是引用计数算法对于对象之间相互循环引用问题难以解决，因此java并没有使用引用计数算法。


# 根搜索算法
通过一系列的名为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连，则证明该对象是不可用的,垃圾收集器将回收其所占的内存。
主流的商用程序语言C#、java和Lisp都使用根搜素算法进行内存管理。
Java语言中以下可作为GC Roots：
- 虚拟机栈（栈帧中的本地变量表）中引用的对象
- 方法区中的类静态属性引用的对象
- 方法区中的常量引用的对象
- 本地方法栈中JNI的引用的对象

# Java中常用的垃圾收集算法

## 标记-清除算法：
最基础的垃圾收集算法，算法分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成之后统一回收掉所有被标记的对象。

标记-清除算法的缺点有两个：首先，效率问题，标记和清除效率都不高。其次，标记清除之后会产生大量的不连续的内存碎片，空间碎片太多会导致当程序需要为较大对象分配内存时无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

## 复制算法：
将可用内存按容量分成大小相等的两块，每次只使用其中一块，当这块内存使用完了，就将还存活的对象复制到另一块内存上去，然后把使用过的内存空间一次清理掉。这样使得每次都是对其中一块内存进行回收，内存分配时不用考虑内存碎片等复杂情况，只需要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。

复制算法的缺点显而易见，可使用的内存降为原来一半。

## 标记-整理算法：
标记-整理算法在标记-清除算法基础上做了改进，标记阶段是相同的标记出所有需要回收的对象，在标记完成之后不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，在移动过程中清理掉可回收的对象，这个过程叫做整理。

标记-整理算法相比标记-清除算法的优点是内存被整理以后不会产生大量不连续内存碎片问题。

复制算法在对象存活率高的情况下就要执行较多的复制操作，效率将会变低，而在对象存活率低的情况下使用标记-整理算法效率会大大提高。

## 分代收集算法：
根据内存中对象的存活周期不同，将内存划分为几块，java的虚拟机中一般把内存划分为新生代和年老代，当新创建对象时一般在新生代中分配内存空间，当新生代垃圾收集器回收几次之后仍然存活的对象会被移动到年老代内存中，当大对象在新生代中无法找到足够的连续内存时也直接在年老代中创建。

**现在的Java虚拟机就联合使用了分代、复制、标记-清除和标记-整理算法**


# 引用分类
- 强引用：Object o = new Object()这种引用，只要强引用存在，垃圾收集器就不会回收
- 软引用：一些还可用，但并非必需的对象。在系统将要发生内存溢出异常之前，将会把这些对象列入回收范围。通过SoftReference实现。
- 弱引用：非必须对象。比软引用更弱。只能生存到下一次垃圾回收之前，通过WeakReference实现。
- 虚引用：幽灵引用，最弱的一种引用。一个对象是否有虚引用，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用的唯一目的就是希望能在对象在被垃圾收集时得到一个系统通知。通过PhantomReference实现。

# 二次标记与finalize()方法
在根搜索算法中不可达的对象，将会被第一次标记并根据是否实现了finalize方法和是否调用过finalize筛选，如果这个对象认为有必要执行finalize方法，就会放入一个F-Queue的队列中执行，稍后GC将对F-Queue中的对象进行二次标记，如果对象在finalize中拯救了自己（只要重新与引用链上任何对象建立关联就行），那么二次标记后他将移出“即将回收”队列，如果没有，就离死亡不远了。不过finalize只会执行一次，下次回收时就无法拯救了。（不建议实现finalize方法）
