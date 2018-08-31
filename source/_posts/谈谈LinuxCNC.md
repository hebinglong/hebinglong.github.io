---
title: 谈谈LinuxCNC
date: 2018-08-16 10:05:23
tags:
---

# 谈谈LinuxCNC

## 背景

​	在最开始打算摸索一下LinuxCNC是因为ROS一般用于科研，工业上较少使用，究其根本是因为其实时性与稳定性达不到工业标准。而LinuxCNC项目本身就是向工业方向发展，其可靠性与稳定性应该是没有问题的。故希望从LinuxCNC项目学习，主要包括保证系统实时性的方式、系统架构（整个控制系统中各个模块的解耦方式）、以及在线实时轨迹规划算法。在搜索资料发现，LinuxCNC项目原名**EMC2**，同时有另外一个分支叫**Machinekit**.

LinuxCNC项目官方网址：http://linuxcnc.org/

Machinekit项目官方网址：http://www.machinekit.io

直到目前为止（2018年8月16日），LinuxCNC的官方稳定版本号为**2.7.14**，Machinekit没有明确的版本号，但从推荐的镜像介绍里面，使用的是基于3.8内核的Debian with xenomai kernel ，同时也表示，基于rt-preempt 4.x.x kerneled versions将会很快推出。

​	在这里先介绍一下Machinekit的背景，Machinekit项目最开始是希望在BeagleBoneBlack（俗称BBB，TI发布的一款嵌入式板，类似于树莓派，但是性能比树莓派稍弱，但是毕竟是TI出品，元器件与Layout都是工业级别的，稳定性有保证，相比之下，树莓派性能是比较强，但是貌似稳定性跟BBB不是一个级别，无法应用在工业领域）上移植LinuxCNC实现3D打印，无奈LinuxCNC项目最初使用的实时方案是RTAI，而RTAI不支持ARM平台，无奈之下Machinekit的作者只好自己移植其他的实时内核。所以总结起来就是 Machinekit更像是针对BBB这款硬件使用的移植版LinuxCNC。

## LinuxCNC与Machinekit的实时方案比较

查阅两者官网可知，目前LinuxCNC项目支持的实时方案为

参考：http://linuxcnc.org/docs/2.7/html/getting-started/getting-linuxcnc.html

{% asset_img LinuxCNC_rt.png LinuxCNC实时方案 %}

Machinekit项目支持的实时方案为：

参考：http://www.machinekit.io/docs/common/UnifiedBuild/

> support for Xenomai and RT-PREEMPT realtime threads besides RTAI
>
> There should be minimal user configuration changes for using the new RT options.
>
> kernel autodetection
>
> The 'unified build' branch will detect the RT features of the running kernel and choose an appropriate thread flavor.
>
> runtime loading of support modules
>
> All thread-specific code has been wrapped into shared objects and libraries which are loaded on demand. This enables fixes, upgrades or tests by just exchanging a file.



总结一下就是

LinuxCNC支持Ubuntu 与Debian，实时方案选择Preempt-RT和RTAI

Machinekit对操作系统没有明确的限制，但是官方在Debian上测试没问题，相比之下，Machinekit支持Xenomai 、RT-PREEMPT实时方案。

对比LinuxCNC与Machinekit发现，Machinekit更致力于一套代码在多套平台上使用，其实现方式为抽象RTAPI层作为实时方案的抽象，在运行时动态加载对应的库文件。

个人感觉，Machinekit的格局比LinuxCNC要大，支持面更广。但又因为其通用性，可能在稳定性上比不上LinuxCNC，当然这只是个人推测，并没有实际测试。当然，假如只是为了学习，那么Machinekit应该更有意思。

## 软件架构

大概浏览了一下LinuxCNC 与 Machinekit的开发文档，发现两者的整体架构非常一致（那当然啦，毕竟本是一家），而LinuxCNC 的文档看起来好像比Machinekit更加详细。下面就先以LinuxCNC 的软件架构进行分析，最后在对比Machinekit，看看Machinekit是如何在LinuxCNC的基础上做到多平台兼容的。



> 在写这章之前 笔者希望先了解一下xenomai的使用方式，故该博文先暂停。



## 软PLC Classic Ladder

