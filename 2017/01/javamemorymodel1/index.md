# Java 内存模型_1

## 概述

[本文记录 Java 中的内存模型的基础部分1。](http://www.infoq.com/cn/articles/java-memory-model-1)
本篇作为学习 Java 内存模型基础部分的笔记,加上些许自己的理解和解释.

<!-- more -->

### 为什么需要理解 Java 内存模型

结论:并发产生的内存可见性问题.

并发编程中的两个关键问题:

- 线程之间如何通行(以何种机制来交换信息)
- 线程之间如何同步


已有的通信机制:

- 共享内存: 线程之间共享程序的公共状态,线程之间通过读写内存中的公共状态来隐式进行通信.
- 消息传递: 在这种模型中,线程之间没有公共状态,线程之间必须通过发送消息显式地通信.


同步,指的是控制不同线程之间操作发生的相对顺序的机制:
- 在共享内存模型中,程序员必须显式的指定某个方法或者某段代码需要在线程之间互斥执行.
- 在消息传递的模型里,由于消息的发送必须在消息的接收之前,因此,同步是隐式进行的.


java 采用的是共享内存模型,而 java 线程之间的通信总是隐式进行,整个通信过程对程序员完全是透明的.
对于 java 程序员而言,如果不了解 java 内存模型,在编写多线程程序的时候,就会遇到各种各样的内存可见性的问题.所以,对 java 的内存模型需要有一定的了解.


### Java 采用的共享内存模型是什么样的

> Java 采用的是共享内存模型作为线程间的通信机制.

共享内存: 堆内存,在 java 中,所有的 **实例域,静态域和数组元素** 存储在堆内存中,堆内存在线程之间共享.
局部变量,方法定义的参数和异常处理参数,不会在线程之间共享,它们不会有内存可见性问题,也不受内存模型的影响.

Java 线程之间的通信由 Java内存模型(JMM)控制,JMM 决定了一个线程对共享变量的写入何时对另个线程可见.从抽象的角度来看,JMM 定义了线程和主内存之间的抽象关系:线程之间的**共享变量存储在主内存中**,每个线程都有一个私有的本地内存,本地内存中存储了该线程以读写共享变量的副本.**本地内存是 JMM 的一个抽象概念,并不真实存在**.它涵盖了缓存,写缓冲区,寄存器以及其他的硬件和编译器优化.

![Java 内存模型抽象示意图](https://static001.infoq.cn/resource/image/b0/9b/b098a84eb7598d70913444a991d1759b.png)


分析: 线程 A 和线程 B 通信过程
1. 线程 A 将本地内存 A 中更新过的共享变量刷新到主内存中去.
2. 线程 B 去主内存中读取线程 A 更新过的共享变量.


![内存模型案例](https://static001.infoq.cn/resource/image/2c/cb/2c452d147bf0d09b14b770d3990740cb.png)

本地内存 A 和本地内存 B 都有主内存中共享的变量 x 的副本.假设初始时,这个三个内存中的 x 的值都是 0.线程 A 在执行时,把已经更新的 x 值(假设为1) 临时存放在自己的本地内存 A 中. 当线程 A 和线程 B 需要通信的时,线程 A 首先会把本地内存中修改后的 x 值刷新到主内存中,此时主内存中的 x 值变为 1.随后,线程 B 到主内存中读取线程 A 更新之后的 x 值,此时线程 B 的本地内存的 x 值也变为 1.

从整体上看,这两个步骤实质上是线程 A 在向线程 B 发送消息,而且这个通信过程必须经过主内存.JMM 通过控制主内存与每个线程的本地内存之间的交互,来为 java 程序员提供内存可见性保证.

### 重排序(即 java 如何实现同步)
##### 为什么需要重排序
在执行程序的时候,为了提高性能,**编译器 和 处理器** 常常会对指令做重排序.有三种重排序.
##### 什么是重排序
1. 编译器优化重排序(编译器重排序).编译器在不改变单线程**程序语义**的前提下,重新安排语句的**执行顺序**.
2. 指令级并行重排序(处理器重排序).现代处理器采用**指令级并行**技术来执行多条指令**重叠执行**.在不存在**数据依赖**的前提下,处理器可以改变语句对应的指令的执行顺序.
3. 内存系统的重排序(处理器重排序).由于处理器使用缓存和读写缓冲区,这使得加载和存储操作看上去可能是在错乱执行.
所以,从 java 源代码到执行阶段,会一次经过三个重排序.以上所述的三个重排序,一个属于编译器重排序,两个属于处理器重排序.


##### 如何避免重排序带来的问题
三个重排序,都可能导致多线程出现内存可见性的问题.
###### 一个例子,说明重排序产生的问题
![处理器指令](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/JMMprocessor_1.png?raw=true)

假设处理器 A 和处理器 B 按程序的顺序并行执行内存访问，最终却可能得到 x = y = 0 的结果。具体的原因如下图所示：

![程序的顺序](https://static001.infoq.cn/resource/image/90/df/9026b8f4b6c1fae4270615e0aadc7cdf.png)

1. 这里处理器 A 和处理器 B 可以同时执行赋值操作 A1,B1,将共享变量写入各自的写缓冲区（缓冲区 A ，缓冲区 B）,此时处理器 A 中的写缓冲区中的变量 a 值为1,处理器 B 写缓冲区中的变量 b 值为2,但是还未刷新到内存中;
2. 然后执行 A2,B2 操作,从内存中读取变量 b 和变量 a 的值(此时的变量 a 和变量 b 的值还是初始状态,都为0),并赋值给 x 和 y,所以,此时的 x 和 y 的值都为 0.
3. 最后执行 A3 和 B3 操作，把处理器 A  和 处理器 B 写缓存区中保存的脏数据刷新到内存中,此时内存中的变量 a 和变量 b 就是 1 和 2 了。
4. 执行完所有操作之后, x = y = 0 .

从内存操作实际发生的顺序来看，直到处理器 A 执行 A3 来刷新自己的写缓存区，写操作 A1 才算真正执行了。虽然处理器 A 执行内存操作的顺序为：A1->A2，但内存操作实际发生的顺序却是：A2->A1。此时，处理器 A 的**内存操作顺序**被重排序了（处理器 B 的情况和处理器 A 一样，这里就不赘述了）。

这里的关键是，由于写缓冲区仅对自己的处理器可见，它会导致处理器执行内存操作的顺序可能会与内存实际的操作执行顺序不一致。由于现代的处理器都会使用写缓冲区，因此现代的处理器都会允许对写-读操作重排序。
对于编译器, JMM 的编译器重排序规则会禁止特定类型的**编译器重排序**(并不是所有的编译器重排序都需要被禁止的).
对于处理器, JMM 的处理器重排序规则会要求 java 编译器在生成指令序列的时候,插入指定类型的**内存屏障指令**,通过内存屏障指令来禁止特定类型的处理器重排序(不是所有的处理器重排序都要禁止的).

###### 处理器重排序

> 现代处理器使用写缓冲区来临时保存向内存写入的数据.写缓冲区可以保障指令流水线持续运行,它可以避免由于处理器停顿下来等待向内存写入数据而产生延迟.同时通过以批处理的方式刷新写缓冲区,以及合并写缓冲区中对同一个内存地址的多次写,可以减少对内存总线的占用.

缓冲区的这一特性是可以加速程序的运行,然而每个处理器的写缓冲区,仅仅对它所在的处理器可见.这个特性会对内存操作的执行顺序产生重要的影响:处理器对内存的读写操作的执行顺序,不一定与内存实际发生的读写顺序一致.

![重排序类型](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/JMM_ReOrder.png?raw=true)

上表单元格中的 “N” 表示处理器不允许两个操作重排序，“Y” 表示允许重排序。
从上表我们可以看出：常见的处理器都允许 Store-Load 重排序；常见的处理器都不允许对存在数据依赖的操作做重排序。sparc-TSO 和 x86 拥有相对较强的处理器内存模型，它们仅允许对写-读操作做重排序（因为它们都使用了写缓冲区）。

1. 注1：sparc-TSO 是指以 TSO(Total Store Order) 内存模型运行时，sparc 处理器的特性。
2. 注2：上表中的 x86 包括 x64 及 AMD64。
3. 注3：由于 ARM 处理器的内存模型与 PowerPC 处理器的内存模型非常类似，本文将忽略它。
4. 注4：数据依赖性后文会专门说明。

###### 内存屏障指令

为了保证内存可见性，java 编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。JMM 把内存屏障指令分为下列四类：
![内存屏障](https://github.com/tinggengyan/tinggengyan.github.io/blob/source/imgur/JMM_Barriers.png?raw=true)

StoreLoad Barriers 是一个“全能型”的屏障，它同时具有其他三个屏障的效果。现代的多处理器大都支持该屏障（其他类型的屏障不一定被所有处理器支持）。执行该屏障开销会很昂贵，因为当前处理器通常要把写缓冲区中的数据全部刷新到内存中（buffer fully flush）。

####### happens-before

从 JDK5 开始，java 使用新的 JSR -133 内存模型,JSR-133 使用 happens-before 的概念来阐述操作之间的内存可见性。在 JMM 中，如果一个操作执行的**结果**需要对另一个操作可见，那么这两个操作之间必须要存在 happens-before 关系。这里提到的两个操作既可以是在一个线程之内，也可以是在不同线程之间。
与程序员密切相关的 happens-before 规则如下：
* 程序顺序规则：一个线程中的每个操作，happens-before 于该线程中的任意后续操作。（注解：如果只有一个线程的操作，那么前一个操作的结果肯定会对后续的操作可见。)
* 监视器锁规则：对一个监视器锁的**解锁**，happens-before 于随后对这个监视器锁的**加锁**。
* volatile 变量规则：对一个 volatile 域的写，happens-before 于任意后续对这个 volatile 域的读。
* 传递性：如果 A happens-before B，且 B happens-before C，那么 A happens-before C。

**注意**，两个操作之间具有 happens-before 关系，并不意味着前一个操作必须要在后一个操作之前执行！happens-before 仅仅要求前一个操作（**执行的结果**）对后一个操作可见，且前一个操作按顺序排在第二个操作之前(the first is visible to and ordered before the second).

![happens-before与JMM的关系](http://ifeve.com/wp-content/uploads/2013/01/552.png)
happens-before 与 JMM 的关系.

如上图所示，一个 happens-before 规则通常对应于多个编译器和处理器重排序规则。对于 java 程序员来说，happens-before 规则简单易懂，它避免 java 程序员为了理解 JMM 提供的内存可见性保证而去学习复杂的重排序规则以及这些规则的具体实现。



### 感激,非常感激，万分的感激！
感谢以下的文章以及其作者和翻译的开发者们,排名不分先后
* [深入理解Java内存模型（一）——基础](http://www.infoq.com/cn/articles/java-memory-model-1)
