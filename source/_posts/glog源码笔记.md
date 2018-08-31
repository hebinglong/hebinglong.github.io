---
title: glog源码笔记
date: 2018-08-28 13:41:54
tags:
---

异步信号安全

## 背景知识

### RTTI

​	RTTI（Run-Time Type Identification)，通过运行时类型信息程序能够使用基类的指针或引用来检查这些指针或引用所指的对象的实际派生类型。

### Name Mangling概述

​	我们假设读者都有这样一个背景知识：“程序员写了一个叫functionName()的函数，该源程序编译为可执行程序后，这个程序的入口不在叫functionName(),而是一些看不懂的如abcde()的函数名”。Name Mangling就是编译过程中"functionName"变为“abcde”的过程，它的逆过程一般称为 demangling。

​	Name Mangling本身和glog没有什么关系，但是有时候我们在**程序异常崩溃的时候希望通过堆栈信息查找程序异常的位置**的时候，由于Name Mangling的原因，只能知道问题出在了“abcde”这个函数里，此时开发者就心里一万只草泥马了，我根本没有写过“abcde”这个函数啊，怎么会报错呢？**那么在这波如此反人类的操作中，日志模块有必要实现demangling这个过程**。下面引用一段网上摘抄关于Name Mangling的叙述：

> ​	大型程序是通过多个模块构建而成，模块之间的关系由makefile来描述。对于由C++语言编制的大型程序而言，也是符合这个规则。
> 	程序的构建过程一般为：各个源文件分别编译，形成目标文件。多个目标文件通过链接器形成最终的可执行程序。显然，从某种程度上说，编译器的输出是链接器的输入，链接器要对编译器的输出做二次加工。**从通信的角度看，这两个程序需要一定的协议来规范符号的组织格式。这就是Name Mangling产生的根本原因。**
> 	C++的语言特性比C丰富的多，C++支持的函数重载功能是需要Name Mangling技术的最直接的例子。对于重载的函数，不能仅依靠函数名称来区分不同的函数，因为C++中重载函数的区分是建立在以下规则上的：
> 函数名字不同 || 参数数量不同||某个参数的类型不同
> 那么区分函数的时候，应该充分考虑参数数量和参数类型这两种语义信息，这样才能为却分不同的函数保证充分性。
> 	当然，C++还有很多其他的地方需要Name Mangling，如namespace, class, template等等。
> 总的来说，**Name Mangling就是一种规范编译器和链接器之间用于通信的符号表表示方法的协议，其目的在于按照程序的语言规范，使符号具备足够多的语义信息以保证链接过程准确无误的进行.**

### 如何识别C++编译以后的函数名（demangle）

> Transforming C++ ABI identifiers (like RTTI symbols) into the original C++ source identifiers is called “demangling.”

C/C++语言在编译以后，函数的名字会被编译器修改，改成编译器内部的名字，这个名字会在链接的时候用到。如果用[backtrace](http://www.gnu.org/software/libc/manual/html_node/Backtraces.html)之类的函数打印堆栈时，显示的就是被编译器修改过的名字，比如说_Z3foov 。 那么这个函数真实的名字是什么呢？

每个编译器都有一套自己内部的名字，这里只是针对linux下g++而言。
以下是基本的方法:
每个方法都是以_Z开头，对于嵌套的名字（比如名字空间中的名字或者是类中间的名字,比如Class::Func）后面紧跟N ， 然后是各个名字空间和类的名字，每个名字前是名字字符的长度，再以E结尾。(如果不是嵌套名字则不需要以E结尾)

比如上面的_Z3foov 就是函数foo() , v 表示参数类型为void .
又如N:C:Func 经过修饰后就是 _ZN1N1C4FuncE, 这个函数名后面跟参数类型。 如果跟一个整型，那就是_ZN1N1C4FuncEi

另外在linux下有一个工具可以实现这种转换，这个工具是c++filt , 注意不是c++filter.

```shell
xuyang@ubuntu15:~/blog$ c++filt _ZN1N1C4FuncEi
N::C::Func(int)
```



## 不可重入、线程安全与异步信号安全 

http://www.cnblogs.com/zhaoyl/archive/2012/10/03/2711018.html

https://www.ibm.com/developerworks/cn/linux/l-reent.html

## glog中的辅助调试手段



```c++
// We don't use assert() since it's not guaranteed to be
// async-signal-safe.  Instead we define a minimal assertion
// macro. So far, we don't need pretty printing for __FILE__, etc.
/*in symbolize.cc*/
```

```c++
// The demangler is implemented to be used in async signal handlers to
// symbolize stack traces.  We cannot use libstdc++'s
// abi::__cxa_demangle() in such signal handlers since it's not async
// signal safe (it uses malloc() internally).
/*in demangle.h*/
```



## glog中的宏



## 利用匿名对象



## glog在实时系统的改造