---
title: Linux操操作系统实时性分析
date: 2018-08-15 16:32:19
tags:
---

# Linux操操作系统实时性分析

## 博主总结

本文摘录自 https://blog.csdn.net/lu_embedded/article/details/53572620

​	该文章是博主在摸索实时Linux实现方案时搜索到的，在知网上关于标准Linux实时性的分析与原博文相差无几，故直接摘录。在进一步深入摸索中，希望对比 RT-Preempt与Xenomai两种实时方案有何异同时找到该文章，对这两种实时改造方案进行了分析。

​	总结如下：

- RT-Preempt属于直接修改标准linux内核的源码，使得内核可抢占，达到实时要求。

- Xenomai属于双内核实现方式，在标准内核以外构建另外一个实时内核，将标准内核作为实时内核中的一个进程来运行。

- 在开发流程上，RT-Preempt提供POSIX标准接口，与一般linux程序开发无异；Xenomai除了提供POSIX接口以外，还提供其他一些商用实时系统兼容的接口，方便用户移植代码。

- 在上述两种实时方案中进行选择时，官方给出的答案如下：

  Xenomai 3 FAQ

  **Q**: I can run POSIX based applications directly over a PREEMPT_RT kernel on my target system, so what is the point of running Xenomai 3 there?

  **A**: If your application is already fully POSIXish, and the performances requirements are met, then there is likely no point. However, you may want to consider Xenomai 3 in two other situations:

  - you want to port a legacy embedded application to Linux without having to switch APIs, i.e. you don't want to rewrite it on top of the POSIX interface. Xenomai may help in this case, since it supports multiple programming interfaces over a common real-time layer, including emulators of traditional RTOS APIs. Xenomai 3 will make those APIs available to a PREEMPT_RT based system as well.


  - the target hardware platform has limited horsepower, and/or you want the real-time job to put the smallest possible overhead on your system. This is where dual kernels are usually better than a native preemption system. With the latter, all parts of the Linux system have to run internal code that prevents real-time activities from being delayed in an unacceptable manner (e.g. priority inheritance mechanism, threaded IRQ handlers). In a dual kernel system, there is no need for this, since the real-time co-kernel runs separately from the normal Linux kernel. Therefore, regular Linux activity is not charged for real-time activity, it does not even have to know about it.

    ​

  In short, there cannot be any pre-canned answer to such a question: it really depends on your performance requirements, and your target hardware capabilities. This has to be evaluated on a case-by-case basis. Telling the world about "we can achieve X microseconds worst-case latency" without specifying the characteristics of the target platform would make no sense. 

  ​

## 制约标准Linux实时性的因素

​	虽然Linux系统功能强大、实用性强、易于软件的二次开发，并且提供编程人员熟悉的标准API。但是由于Linux系统一开始就被设计成GPOS（通用操作系统），它的目的是构建一个完整、稳定的开源操作系统，尽量缩短系统的平均响应时间，提高吞吐量，注重操作系统的整体功能需求，达到更好地平均性能。（在操作系统中，我们可以把吞吐量简单的理解为在单位时间内系统能够处理的事件总数。） 
　　因此在设计Linux的进程调度算法时主要考虑的是公平性，也就是说，调度器尽可能将可用的资源平均分配给所有需要处理器的进程，并保证每个进程都得以运行。但这个设计目标是和实时进程的需求背道而驰的，所以**标准Linux并不提供强实时性**。 
　　Linux系统实时性不强使其在嵌入式应用中有一定的局限性，主要是受内核可抢占性、进程调度方式、中断处理机制、时钟粒度等几个方面的制约，具体如下：

### 进程调度

​	Linux系统提供符合POSIX标准的调度策略，包括FIFO调度策略（`SCHED_FIFO`）、带时间片轮转的实时调度策略（`SCHED_RR`）和静态优先级抢占式调度策略（`SCHED_OTHER`）。Linux进程默认的调度策略为`SCHED_OTHER`，这种调度方式虽然可以让进程公平地使用CPU和其它资源，但是并不能保证对时间要求严格或者高优先级的进程将先于低优先级的执行，这将严重影响系统实时性。那么，将实时进程的调度策略设置为`SCHED_FIFO` 或`SCHED_RR` ，似乎使得Linux系统具备根据进程优先级进行实时调度的能力，但问题在于，Linux系统在用户态支持可抢占调度策略，而在内核态却不完全支持抢占式调度策略。这样运行在Linux内核态的任务（包括系统调用和中断处理）是不能被其它优先级更高的任务所抢占的，由此引起优先级逆转问题。

### 内核抢占机制

​	Linux的系统进程运行分为用户态和内核态两种模式。当进程运行在用户态时，具有高的优先级的进程可以抢占进程，可以较好地完成任务；但是当进程运行在内核态时，即使其他高优先级进程也不能抢占该进程。当进程通过系统调用进入内核态运行时，实时任务必须等待系统调用返回后才能获得系统资源。这和实时系统所要求的高优先级任务运行是相互矛盾的。 
　　当然，这种情况在Linux2.6版本的内核发布以来有了显著改进，**Linux2.6版本后的内核是抢占式的，这意味着进程无论在处于内核态还是用户态，都可能被抢占**。Linux2.6以后的内核提供以下3种抢占模式供用户选择。 
　　`PREEMPT_NONE`——没有强制性的抢占。整体的平均延时较低，但偶尔也会出现一些较长的延时。它最适合那些以整体吞吐率为首要设计准则的应用。 
　　`PREEMPT_VOLUNTARY`——降低延时的第一阶段。它会在内核代码的一些关键位置上放置额外的显示抢占点，以降低延时。但这是以牺牲整体吞吐率为代价的。 
　　`PREEMPT/PREEMPT_DESKTOP`——这种模式使内核在任何地方都是可抢占的，临界区除外。这种模式适用于那些需要软实时性能的应用程序，比如音频和多媒体。这也是以牺牲整体吞吐率为代价的。

### 中断屏蔽

Linux在进行中断处理时都会关闭中断，这样可以更快、更安全地完成自己的任务，但是在此期间，即使有更高优先级的实时进程发生中断，系统也无法响应，必须等到当前中断任务处理完毕。这种状况下会导致中断延时和调度延时增大，降低Linux系统的实时性。

### 时钟粒度粗糙

​	时钟系统是计算机的重要组成部分，相当于整个操作系统的脉搏。系统所能提供的最小时间间隔称为时钟粒度，时钟粒度与进程响应的延迟性是正比关系，即粒度越粗糙，延迟性越长。但时钟粒度并不是越小越好，就同等硬件环境而言，较小的时间粒度会导致系统开销增大，降低整体吞吐率。在Linux2.6内核中，时钟中断发生频率范围是50~1200Hz，周期不小于0.8ms，对于需要几十微秒的响应精度的应用来说显然不满足要求。而在嵌入式Linux系统中，为了提高整体吞吐率，时钟频率一般设置为100HZ或250HZ。 
　　另外，系统时钟负责软定时，当软定时器逐渐增多时会引起定时器冲突，增加系统负荷。

### 虚拟内存管理

​	Linux采用虚拟内存技术，进程可以运行在比实际空间大得多的虚拟空间中。在分时系统中，虚拟内存机制非常适用，然而对于实时系统这是难以忍受的，频繁的页面换进换出会使得系统进程运行无法在规定时间内完成。 
　　对于此问题，Linux系统提供内存锁定功能，以避免在实时处理中存储页被换出。

### 共享资源的互斥访问差异

​	多个任务互斥地访问同一共享资源时，需要防止数据遭到破坏，系统通常采用信号量机制解决互斥问题。然而，在采取基于优先级调度的实时系统中，信号量机制容易造成优先级倒置，即低优先级任务占用高优先级任务资源，导致高优先级任务无法运行。

　　虽然从2.6.12版本之后，Linux内核已经可以在较快的x86处理器上实现10毫秒以内的软实时性能。但**如果想实现可预测、可重复的微秒级的延时，使Linux系统更好地应用于嵌入式实时环境，则需要在保证Linux系统功能的基础上对其进行改造**。下一节将介绍通过实时补丁来提高Linux实时性的方法。

## 常用的实时Linux改造方案

　根据实时性系统要求以及Linux的特点和性能分析，对标准Linux实时性的改造存在多种方法，较为合理的两大类方法为：

1. 直接修改Linux内核源代码。
2. 双内核法。

### 直接修改Linux内核源代码

​	对Linux内核代码进行细微修改并不对内核作大规模的变动，在遵循GPL协议的情况下，直接修改内核源代码将Linux改造成一个完全可抢占的实时系统。核心修改面向局部，不会从根本上改变Linux内核，并且一些改动还可以通过Linux的模块加载来完成，即系统需要处理实时任务时加载该功能模块，不需要时动态卸载该模块。 
　　目前kernel.org发布的主线内核版本还不支持硬实时。为了开启硬实时的功能，必须对代码打补丁。实时内核补丁是多方努力的共同成果，目的是为了降低Linux内核的延时。这个补丁有多位代码贡献者，目前由Ingo Molnar维护，补丁网址如下：www.kernel.org/pub/linux/kernel/projects/rt/。 
　　在配置已经打过实时补丁的内核代码时，我们发现实时补丁添加了第4种抢占模式，称为`PREEMPT_RT`（实时抢占）。实时补丁在Linux内核中添加了几个重要特性，包括使用可抢占的互斥量来替代自旋锁；除了使用`preempt_disable()`保护的区域以外，内核中的所有地方都开启了非自愿式抢占（involuntary preemption）功能。这种模式能够显著降低抖动（延时的变化），并且使那些对延时要求很高的实时应用具有可预测的较低延时。 
　　这种方法存在的问题是：很难百分之百保证，在任何情况下，GPOS程序代码绝不会阻碍RTOS的实时行为。也就是说，通过修改Linux内核，难以保证实时进程的执行不会遭到非实时进程所进行的不可预测活动的干扰。

### 双内核法

实际上，双内核的设计缘由在于，人们不相信标准Linux内核可以在任何情况下兑现它的实时承诺，因为GPOS内核本身就很复杂，更多的程序代码通常会导致更多的不确定性，这样将无法符合可预测性的要求。更何况Linux内核极快的发展速度，使其会在很短的时间内带来很大的变化，直接修改Linux内核源代码的方法将难以保持同步。 
　　双内核法是在同一硬件平台上采用两个相互配合，共同工作的系统核心，通过在Linux系统的最底层增加一层实时核心来实现。其中的一个核心提供精确的实时多任务处理，另一个核心提供复杂的非实时通用功能。 
　　双内核方法的实质是把标准的Linux内核作为一个普通进程在另一个内核上运行。关键的改造部分是在Linux和中断控制器之间加一个中断控制的仿真层，成为其实时内核的一部分。该中断仿真机制提供了一个标志用来记录Linux的关开中断情况。一般只在修改核心数据结构关键代码时才关中断，所以其中断响应很小。其优点是可以做到硬实时，并且能很方便地实现一种新的调度策略。 
　　为方便使用，实时内核通常由一套可动态载入的模块提供，也可以像编译任何一般的子系统那样在Linux源码树中直接编译。**常用的双内核法实时补丁有RTLinux/GPL、RTAI 和 Xenomai**，其中RTLinux/GPL只允许以内核模块的形式提供实时应用；而RTAI和Xenomai支持在具有MMU保护的用户空间中执行实时程序。下面，我们将对RTAI与Xenomai进行分析。



{% asset_img 1.png RTAI（左）和Xenomai（右）实时内核在Linux中的分层结构 %}

​	图1所示为RTAI和Xenomai两个实时内核分别与标准Linux内核组成双内核系统是的分层结构。可以看到两者有稍微不同的组织形式，与Xenomai让ADEOS掌控所有的中断源不同的是，RTAI拦截它们，使用ADEOS将那些RTAI不感兴趣的中断通知送给Linux（也就是，中断不影响实时时序）。这样混合过程的目的是提高性能，因为在这种情况下，如果中断是要唤醒一个实时任务，就避免了由ADEOS管理中断的开销。从这里可以看出，RTAI的实时性能应该是比Xenomai要好的。 
　　RTAI（Real-Time Linux Application interface）虽然实时性能较好，但对ARM支持不够，更新速度极慢，造成项目开发周期长，研发成本高。 
　　与RTAI相比，Xenomai更加专注于用户态下的实时性、提供多套与主流商业RTOS兼容的API以及对硬件的广泛支持，在其之上构建的应用系统能保持较高实时性，而且稳定性和兼容性更好；此外，Xenomai社区活跃，紧跟主流内核更新，支持多种架构，对ARM的支持很好。 
　　Xenomai是Linux内核的一个实时开发框架。它希望无缝地集成到Linux环境中来给用户空间应用程序提供全面的、与接口无关的硬实时性能。Xenomai是基于一个抽象实时操作系统核心的，可以被用来在一个通用实时操作系统调用的核心上，构建任意多个不同的实时接口。Xenomai项目始于2001年8月。2003年它和RTAI项目合并推出了RTAI/fusion。2005年，因为开发理念不同，RTAI/fusion项目又从RTAI中独立出来作为Xenomai项目。相比之下，**RTAI项目致力于技术上可行的最低延迟**；**Xenomai除此之外还很着重扩展性、可移植性以及可维护性**。Xenomai项目将对Ingo Molnar的`PREEMPT_PT`实时抢占补丁提供支持，这又是与RTAI项目的一个显著的不同。RTAI和Xenomai都有开发者社区支持，都可以作为一个VxWorks的开源替代。 
　　Xenomai是基于Adeos（Adaptive Domain Environment for Operating System）实现的，Adeos的目标是为操作系统提供了一个灵活的、可扩展的自适应环境；在这个环境下，多个相同或不同的操作系统可以共存，共享硬件资源。基于Adeos的系统中，每个操作系统都在独立的域内运行，每个域可以有独立的地址空间和类似于进程、虚拟内存等的软件抽象层，而且这些资源也可以由不同的域共享。与以往传统的操作系统共存方法不同，Adeos是在已有的操作系统下插入一个软件层，通过向上层多个操作系统提供某些原语和机制实现硬件共享。应用上主要是提供了一个用于“硬件-内核”接口的纳内核（超微内核），使基于Linux环境的系统能满足硬实时的要求。 
　　Xenomai正是充分利用了Adeos技术，它的首要目标是帮助人们尽量平缓地移植那些依赖传统RTOS的应用程序到GNU/Linux环境，避免全部重写应用程序。它提供一个模拟器模拟传统实时操作系统的API，这样就很容易移植应用程序到GNU/Linux环境中，同时又能保持很好的实时性。Xenomai的核心技术就是使用一个实时微内核来构建这些实时API，也称作“Skin”。Xenomai通过这种接口变种技术实现了针对多种传统RTOS的应用编程接口，方便传统RTOS应用程序向GNU/Linux的移植。图2描述了Xenomai的这种带Skin的分层架构。

{% asset_img 2.png 带Skin接口的Xenomai分层结构 %}

​	从图2可以看出，Xenomai系统包含多个抽象层：Adeos纳内核直接工作在硬件之上；位于Adeos之上的是与处理器体系结构相关的硬件抽象层（Hardware Abstraction Layer, HAL）；系统的中心部分是运行在硬件抽象层之上的抽象的实时内核，实时内核实现了一系列通用RTOS的基本服务。这些基本服务可以由Xenomai的本地API（Native）或由建立在实时内核上的针对其他传统RTOS的客户API提供，如RTAI、POSIX、VxWorks、uITRON、pSOS+等。客户API旨在兼容其所支持的传统RTOS的应用程序在Xenomai上的移植，使应用程序在向Xenomai/Linux体系移植的过程中不需要完全重新改写，此特性保证了Xenomai系统的稳健性。Xenomai/Linux系统为用户程序提供了用户空间和内核空间两种模式，前者通过系统调用接口实现，后者通过实时内核实现。用户空间的执行模式保证了系统的可靠性和良好的软实时性，内核空间程序则能提供优秀的硬实时性。