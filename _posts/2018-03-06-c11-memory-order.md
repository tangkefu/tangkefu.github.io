---
layout: post
title: "C++11 memory order学习"
description: ""
category: "cpp"
tags: [c++]
---
{% include JB/setup %}


参考《计算机体系机构: 量化方法研究》, memory consitency是由chche coherence引出的
  
>图1

![p1](https://m.lmoar.com/static/blog/cpp_blog_1.jpg)

图1描述的问题是:

  1. 处理器A缓存(读)X的值1;
  2. 处理器B缓存(读)X的值1;
  3. 处理器A直写(write through) X的值为0;
  4. 处理器B中缓存的是旧的X值(1)

> 通俗的说, 如果在每次读取某一数据项时都会返回该数据项的最新写入值，则称这个存储系统是coherence(一致)的,  主要包含了内存系统两方面的行为表现
> 
  Coherence定义了一个读操作能获得什么样的值
  Consitency定义了何时一个写操作的值会被读操作获得

> 简单来说，coherence其实保证的就是对某一个地址的读操作返回的值一定是那个地址的最新值(注意coherence和consistency最大的区别就是是对某一个地址还是全局)，而这个最新值可能是该线程所处的CPU核心刚刚写进去的那个最新值，也可能是另一个CPU核心上的线程刚刚写进去的最新值

>

具体严谨的定义
![p2](https://m.lmoar.com/static/blog/cpp_blog_2.jpg)

> 这三个条件只是保证了coherence，还未考虑consistency的问题。比如，如果一个处理器对X的写入操作仅比另一个处理器对X的读取操作提前很短的一点时间，那就不可能确保该读取操作会返回这个写入值。这个写入值多久后能确保被读取操作读取到，这正是memory consistency讨论的问题。
> 

#### Memory consistency model
  本质上, 内存一致性模型限制了读操作的返回值
  直观上, 读操作应该返回“最后” 一次写入的值

* 单处理器系统, "最后"由程序次序定义
* 多处理器系统, 称之为顺序连贯(sequential consistency, SC) 

##### 顺序连贯(Sequential consistency)
>  
* 在每个处理器内，维护程序次序
* 在所有处理器间维护一个单一的操作次序，即所有处理器看到的操作次序要一样。这就使内存操作需要有原子性(或瞬发性),可以想象系统由单一的全局内存组成，每一时刻，由switch将内存连向任意的处理器，每个处理器按程序顺序发射(issue)内存操作。这样，switch就可以提供全局的内存串行化性质

> 一致性维护方法
![sc](https://m.lmoar.com/static/blog/cpp_blog_3)

图 a 阐述了 SC 对程序次序的要求
> 
> 当处理器 P1想要进入临界区时,
它先将 Flag1 更新为 1, 再去判断 P2 是否尝试进入临界区(Flag2). 

> 若 Flag2 为 0,
表示 P2 未尝试进入临界区,此时 P1 可以安全进入临界区。

> 这个算法假设如果 P1 中读到 Flag2 等于0, 那么P1的写操作(Flag1=1)会在P2
的写操作和读操作之前发生,这可以避免 P2 也进入临界区。SC 通过维护程序次序来保证这一点。

图 b 阐述了原子性要求。
> 
> 原子性允许我们假设 P1 对 A 的写操作对整个系统(P2, P3) 是同时可见的：P1 将 A 写成1; P2 先看到 P1 写 A 后才写 B;P3 看到 P2 写 B 后才去读 A, P3 此时读出的 A 必须要是 1 （而不会是0）因为从 P2 看来操作执行次序为 (A=1)->(B=1), 不允许P3在看到操作 B=1 时，没有看到 A=1.
> 
 
实现SC需要付出代价，使性能降低，同时也会限制编译器的优化。
  
1. 在无缓存的体系结构, 以三个硬件优化为例
  
  * 带有读旁路的写缓冲(write buffers with read bypassing)
    
    > 读操作可以不等待写操作，导致后续的读操作越过前面的写操作，违反程序次序

  * 重叠写(Overlapping writes) 

    > 对于不同地址的多个写操作同时进行，导致后续的写操作越过前面的读操作，违反程序次序
  * 非阻塞读(Nonblocking reads)

    > 多个读操作同时进行，导致后续的读操作越过前面的读操作先执行，违反程序次序

  为解决上述硬件优化对SC带来的问题，提出了程序次序要求(program order requirement): 
  
  * 按照程序次序，处理器的上一次内存操作需要在下一次内存操作前完成
  
2. 在有缓存的体系结构下实现SC
  
  对于带有缓存的体系结构，这种数据的副本（缓存）的出现引入了三个额外的问题
  
  * 缓存一致性协议(cache coherence protocols)

    * 一个写操作最终要对所有处理器可见
    * 对同一地址的写操作串行化

  * 检查写完成(detecting write completion)
  * 维护写原子性(maintaining write atomicity)
    * “将值的改变传播给多个处理器的缓存”这一操作是非原子操作（非瞬发完成的），因此需要采取特殊措施提供写原子性的假象

注意到，与特定硬件优化，如重叠写(overlapping writes)、缓存(cache)等技术相结合时，要保证这种SC是有代价的

为获得好的性能, 根据《量化》一书，relaxed memory consistency models有多种模型, A->B表示 A 先于 B 完成, 比如放松 W->R；放松W->W；放松 R->W 和 R->R，这包括多种模型，其中Release consistency我们单独拿出来说
> 

#### Release consistency
> Release consistency包含两个同步操作，acquire和release
> 
* ACQUIRE:  对于所有其它参与者来说, 在此操作后的所有读写操作必然发生在ACQUIRE这个动作之后
* RELEASE:  对于所有其它参与者来说, 在此操作前的所有读写操作必然发生在RELEASE这个动作之前

注意, 这其中任意一个操作, 都只保证了一半的顺序:
对于ACQUIRE来说, 并没保证ACQUIRE前的读写操作不会发生在ACQUIRE动作之后.
对于RELEASE来说, 并没保证RELEASE后的读写操作不会发生在RELEASE动作之前.
因此release和acquire的组合形成了内存屏障

#### C++11 memory order
##### synchronized-with 与 happens-before 关系 
线程之间的关系

1. 简单地说,如果线程 A 写了变量 x, 线程 B 读了变量 x, 那么我们就说线程 A,
B 间存在 synchronized-with 关系
2. Happens-before 指明了哪些指令将看到哪些指令的结果

   > 2.1 对于单线程而言,这很明了
  
   > 2.2 对于多线程而言,如果一个线程中的操作 A inter-thread happens-before 另一个线程中的操作 B, 那么 A happens-before B
  
  Inter-thread happens-before 概念相对简单,并依赖于 synchronized-with 关系:
如果一个线程中的操作 A synchronized-with 另一个线程中的操作 B, 那么 A inter-thread happens-before B. Inter-thread happens-before 关系具有传递性


C++11  述了 6 种可以应用于原子变量的内存次序

1. `memory_order_relaxed`
2. `memory_order_consum`
3. `memory_order_acquire`
4. `memory_order_release`
5. `memory_order_acq_rel`
6. `memory_order_seq_cst`

虽然共有 6 个选项,但它们表示的是三种内存模型

1. sequential consistent(`memory_order_seq_cst`)
2. relaxed(`memory_order_relaxed`)
3. acquire release ( rest ...)

##### `sequential consisten ordering` 顺序一致次序

对应于 `memory_order_seq_cst ` 

SC作为默认的内存序，是因为它意味着将程序看做是一个简单的序列。如果对于一个原子变量的操作都是顺序一致的，那么多线程程序的行为就像是这些操作都以一种特定顺序被单线程程序执行

##### `relaxed ordering` 松弛次序
对应于 `memory_order_relaxed`

在原子变量上采用 relaxed ordering 的操作不参与 synchronized-with 关系。在同一线程内对同一变量的操作仍保持happens-before关系，但这与别的线程无关

在 relaxed ordering 中唯一的要求是在同一线程中，对同一原子变量的访问不可以被重排
在单个线程内，所有原子操作是顺序进行的。

按照什么顺序？基本上就是代码顺序（sequenced-before）。这就是唯一的限制了！两个来自不同线程的原子操作是什么顺序? 没有定义。。。。

##### 获取-释放次序(acquire-release ordering)
对应`memory_order_release`, `memory_order_acquire`, `memory_order_acq_rel`

同步对一个变量的读写操作

线程 A 原子性地把值写入 x (release), 然后线程 B 原子性地读取 x 的值（acquire）. 这样线程 B 保证读取到 x 的最新值。

这么一同步，在 x 这个点前后， A, B 线程之间有了个顺序关系，称作 inter-thread happens-before.

[memory_order C++文档说明](http://en.cppreference.com/w/cpp/atomic/memory_order)