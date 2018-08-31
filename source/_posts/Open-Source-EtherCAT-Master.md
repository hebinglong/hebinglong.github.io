---
title: Open Source EtherCAT Master
date: 2018-08-21 14:59:10
tags:
---

##SOME

http://openethercatsociety.github.io

Supported by rt-labs and Speciaal Machinefabriek Ketels

1. **SOEM**：Simple Open EtherCAT Master
2. **SOES**：Simple Open EtherCAT Slave

**SOEM and SOES are small EtherCAT stacks for the embedded market.**

##IgH EtherCAT Master

http://etherlab.org/en/ethercat/



## 总结

1. （网上说的）使用起来SOEM的简单一些，而the IgH EtherCAT® Master更复杂一些，但对EtherCAT的实现更为完整。
2. 好像 Igh 的只支持Linux平台，SOME支持Linux和Windows。
3. 两者都需要实时内核的支持（如： Xenomai, RT-Preempt）
4. 开发时需要注意，不仅对操作系统有要求，对CPU型号、网卡型号也有一定的要求。
5. 为了保证实时性可能还需要修改网卡驱动代码。（网卡驱动可能自带缓冲）

