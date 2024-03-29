---
title: JVM-垃圾回收机制
date: 2022-04-21
draft: false
---



# JVM--垃圾回收机制

​	在之前介绍的 Java 内存运行时区域的各个部分，其中程序计数器、虚拟机栈、本地方法栈 3 个区域随线程而生，随线程而灭，栈中的栈帧随着方法的进入和退出而有条不紊地执行着出栈和入栈的操作。每一个栈帧中分配多少内存基本上是在类结构确定下来时就已知的，因此这几个区域内存分配和回收都具有确定性，在这几个区域内就不需要过多考虑如何回收的问题，当方法结束或者内存结束时，内存自然就回收了。

​	而在 Java 堆和方法区这两个区域则有着显著的不确定性：一个接口的多个实现类需要的内存可能会不一样，一个方法所执行的不同条件分支所需要的内存也可能不一样，只有在运行期间我们才知道程序究竟会创建哪些对象，创建多少个对象，这部分内存的分配和回收是动态的。垃圾收集器所关注的就是这部分内存该如何管理。

## 一、对象已死？

​	我们知道，Java 在诞生之初，就已经确定了自动内存管理的一个亮点，Java 在堆里存放着所有的对象实例，而当程序一步一步执行下去的时候，有一些对象实例可能永远也不会用到了，这个时候就需要 jvm 去自动回收这些内存，方便之后在分配新的对象时，能够有充足的空间。垃圾收集器在对堆进行回收前，第一件事情就是确定堆中的对象哪些还“存活”着，哪些已经“死去”（死去就是指不可能再被任何途径使用的对象）。这里介绍两种算法来确定一个对个对象能否被回收。

### 1. 引用计数算法

引用计数的基本实现思路是,**一个对象加个计数器当用时+1用完-1**,最终回收计数为0的对象.但是这种实现方式有个问题,叫**循环引用**。假设A对象引用了B对象,B对象也引用了A对象。GC在做回收的时候就会觉得两个都不能回收,因为都两个对象都存在引用。

引用计数算法虽然需要占用一些额外的内存空间进行计数，但它的原理是很简单的，判定的效率也高，在大多数情况下都是不错的，所以像 python、微软的COM技术等都采用引用计数算法来进行内存管理。不过Java的各个垃圾收集器都是没有采用这种方法来管理内存的，而是采用无论是空间还是时间效率上更高的可达性**分析算法**来处理。

### 2. 可达性分析算法

当前主流的商用编程语言（Java 、c#等）的内存管理子系统都是通过可达性分析算法来判断对象是否存活。这个算法的基本思路就是通过一系列成为“GC ROOT”的跟对象作为起始节点，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径称为“引用链”。如果某个对象到GC ROOT 之间没有任何引用链相连，则表明这个对象是不可达的，能够被垃圾回收器标记回收的。![引用计数法和可达性分析算法· Issue #44 · xzhuz/blog-gitment · GitHub](https://tva1.sinaimg.cn/large/008i3skNgy1gqyalgsspuj31380ny75i.jpg)

​	而能够作为上述 GC ROOT 的对象包括下面几种：

- 在Java虚拟机栈中引用的对象，比如各个线程中被调用的方法栈里使用到的参数、局部变量、临时变量等。
- 在方法区中类静态属性引用的对象
- 在方法区中常量引用的对象，比如字符串常量常量池里的引用
- 在Java本地方法栈中JNI（即通常所说的Native方法）引用的对象
- Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象等，还有类加载器对象
- 所有被同步锁（synchronized 关键字）持有的对象

### 3. 关于引用

从前面的两个算法来看，判断对象的存活其实和引用是离不开的，但是有一些引用由于其特殊性，在内存充裕的时候，我们希望保留这个对象在内存中，而当内存不够时，我们有能够接受回收掉这部分对象内存的代价，所以，Java对引用的概念进行了扩充，将引用分为了强引用、软引用、弱引用、虚引用。

- 强引用：是最常见，最传统的引用，指程序代码里普遍存在的引用赋值，类似`String s = new String()`;无论在任何情况下只要强引用的关系还存在，垃圾收集器就永远不会回收被引用的对象
- 软引用：用来描述一些有用，但是非必须的对象。只是被软引用关联的对象，在系统要发生内存溢出异常以前，会把这些对象列进回收范围之内进行第二次回收，如果这次回收还是没有足够的内存才会报oom错。利用SoftReference类来实现软引用
- 弱引用：描述非必须的对象，强度比软引用还要弱一点，被弱引用关联的对象只能活在下一次垃圾收集发生为止，当垃圾回收器开始工作，无论内存够不够，这部分内存都会被回收掉。
- 虚引用：基本不算引用，唯一的用处就是对象被回收时，系统会收到一个通知。

## 二、 垃圾回收流程

先介绍一下堆中的分代收集划分：

![JVM中的垃圾回收策略| 大唐札记](https://tva1.sinaimg.cn/large/008i3skNgy1gqybrnbdb0j30e804lglw.jpg)

堆主要分成两个代：

新生代：大部分对象都会在这个区域死去，主要用来存放新生的，在只经历前几次GC的对象

老年代：这里的内存用来存放存活时间比较长的对象

为什么会有这样的划分呢，这就和垃圾回收的算法有关了，下面介绍不同的垃圾回收算法

### 1 、标记-清除(Mark-Sweep)算法

首先识别出所有要回收的对象,然后进行清除

**标记、清除过程效率有限,有内存碎片化问题,不合适特别大的堆**

其他收集算法基本**基于标记-清除的思路**进行改进

![标记复制法、标记清除法和标记整理法的区别_波波的小书房-CSDN博客](https://tva1.sinaimg.cn/large/008i3skNgy1gqybzgkreoj30k10fo3ys.jpg)

内存碎片指的是**内存占用不连续**，比如上面回收过后的内存，在不同的对象之间都留有较小的不连续的内存，如果要新分配一个内存较大的对象，就会造成本身空间够，却无法容纳的问题。基于这样的问题，就出现了标记-整理算法。

### 2、标记-整理算法

类似于标记-清除,但为了避免内存碎片化,它会在**清理过程中将对象移动**,以确保**移动后的对象占用连续的内存空间**，**本质上来说和标记-清除算法是差不多的,但是它多了一个整理的步骤,主要目的也是为了避免内存碎片的问题了**

![JAVA垃圾回收算法——标记-清除算法、复制算法、标记-整理算法、分代收集算法_Simon的博客-CSDN博客](https://tva1.sinaimg.cn/large/008i3skNgy1gqyc32q2wdj30xk0hgdg5.jpg)

但是这样，就需要多遍历一次标记好的存货对象，且还有复制，转移对象内存的操作。

### 3、复制算法

划分两块同等大小的区域,收集时将活着的对象复制到另一块区域.

拷贝过程中将对象顺序存放,就可以避免内存碎片化.**弊端是复制+预留内存会造成一定的内存浪费**

复制到新的内存区域过程中是按顺序放入,所以**复制算法解决了内存碎片的问题**

弊端也很明显,要**预留一块内存作为存活对象的存放区域**.如果说程序运行需要8G的内存,为了避免内存碎片的问题又得加多8G内存用来作复制算法,那就太浪费了，但是复制算法的优点也是明显的，时间效率和空间整齐度都很高，适合那些gc存活数量小的内存区域。事实上，jvm也是这样做的：

![java培训JVM之复制算法(Copying) - 技术聚焦- 尚硅谷](https://tva1.sinaimg.cn/large/008i3skNgy1gqycb4lif1j30e80aq0uc.jpg)

在新生代里，又讲内存分为三个区域：

- **Eden**区 (用于存放新对象)
- **Survivor0**区(用于保存在Eden区中经过垃圾回收后没有被回收的对象)
- **Survivor1**区(用于保存在Survivor0区中经过垃圾回收后没有被回收的对象)

大多数对象诞生都是在Eden区，在初次GC过后就会进入Survivor区，而这里为什么会有两个（0，1）区，则就是为了实现垃圾回收的复制算法。

而在老年代里，由于经过多次GC，这些对象都没有被回收，说明对象的成活率是非常高的，通常就是使用标记-整理方法和标记-清理算法去处理。