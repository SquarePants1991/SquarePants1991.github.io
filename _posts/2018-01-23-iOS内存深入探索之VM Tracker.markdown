---
layout: post
title: iOS内存深入探索之VM Tracker
date: 2018-01-23 16:18:24 +0800
categories: iOS开发
---

# 什么是VM Tracker
VM Tracker是Xcode Instruments自带的一个内存分析工具，可以帮助你快速查看虚拟内存块的用量状态以及根据虚拟内存块的tag进行分类。如果你想知道关于虚拟内存的相关知识，可以先阅读[探索iOS内存分配](http://www.gltech.win/ios%E5%BC%80%E5%8F%91/2018/01/16/%E6%8E%A2%E7%B4%A2iOS%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D.html)这篇文章，如果你对虚拟内存以及VM Region不太了解的话，阅读下面的内容可能会有些障碍。想要使用VM Tracker，使用Instruments的Allocations模版即可。如果模版自带的VM Tracker不显示信息，可以用右边的加号再添加一个VM Tracker。

# VM Tracker列属性解析
![](https://user-gold-cdn.xitu.io/2018/1/23/1612216758e143ac?w=764&h=577&f=png&s=120618)
上面是一个空的iOS App的VM Tracker示意图。一共有9列，下面我来一一解释它们的含义。
* `% of Res`， 当前Type的VM Regions总Resident Size占比。
* `Type`，VM Regions的Type，*All*和*Dirty*算是统计性质的Type，__TEXT表示代码段的内存映射，__DATA表示数据段的内存映射。MALLOC_TINY，MALLOC_LARGE，CG Image等Type可以从VM Region的Extend Info中读取出来，后面会着重介绍。
* `# Regs`，当前Type的VM Region总数。
* `Path`，VM Region是从哪个文件映射过来，因为有些类似于__DATA和mapped file的内存块是从文件直接映射过来的。
* `Resident Size`，使用的物理内存量。
* `Dirty Size`，使用中的物理内存块如果不交换到硬盘保存状态就不能复用，那么就是Dirty的内存块，比如你主动malloc出来的内存块，如果不保留其中的状态就把它给别人用，那你肯定就无法恢复这个内存块的信息，所以它是Dirty的。如果是一个映射到内存的文件，就算使用它的内存块，还是可以重新从磁盘载入文件到内存的，所以是非Dirty的，比如最上面图中的mapped file那一行，你可以看到Dirty Size是0。
* `Swapped Size`, 在OSX中，不活跃的内存页可以被交换到硬盘，这是被交换的大小。在iOS中，只有非Dirty的内存页可以被交换，或者说是被卸载。
* `Virtual Size`，VM Regions所占虚拟内存的大小
* `Res. %`，Resident Size在Virtual Size中的占比

# 使用vm_allocate自定义VM Region
我们可以使用`vm_allocate`方法申请一块虚拟内存。下面是具体代码。
```c
vm_address_t address;
vm_size_t size = 1024 * 1024 * 100;
vm_allocate((vm_map_t)mach_task_self(), &address, size, VM_MAKE_TAG(200) | VM_FLAGS_ANYWHERE);
```
上面的代码申请了一块100M的虚拟内存，`(vm_map_t)mach_task_self()`表示在自己的进程空间内申请。`size`的单位是byte。 `VM_MAKE_TAG(200)`是给你申请的内存块提供一个Tag标记。我这里提供了一个200数值作为标记，后面我会具体介绍这个数值在VM Tracker中的作用。最后我们用VM Tracker看一下我们自己分配的虚拟内存块。
![](https://user-gold-cdn.xitu.io/2018/1/23/16122167590667cb?w=1240&h=483&f=jpeg&s=88631)

你可能会注意到这块内存块的Resident Size和Dirty Size都是0KB，因为我们并没有使用这块内存，所以并没有虚拟内存被关联到物理内存上去。你可以尝试使用这块内存，然后去VM Tracker观察变化。比如使用下面的方式填充内存块。
```c
for (int i = 0; i < 1024 * 1024 * 100; ++i) {
  *((char *)address + i) = 0xab;
}
```

# VM Region的Type
接下来我们来介绍内存块的Type，我曾经思考很久VM Tracker是如何识别出每个内存块的Type的。比如MALLOC_TINY,MALLOC_SMALL,ImageIO等等。答案就在`vm_allocate`方法的最后一个参数flags。flags可以分成2个部分。`VM_FLAGS_ANYWHERE`属于flags里控制内存分配方式的flag，它表示可以接受任意位置的内存分配。它的宏定义如下。
```c
#define VM_FLAGS_ANYWHERE	0x0001
```
从定义可以看出，2个字节就可以存储它，int有4个字节，还剩下2个就可以用来存储标记内存类型的Type了。苹果提供了`VM_MAKE_TAG`宏帮助我们快速设置Type。`VM_MAKE_TAG`实际上做了一件很简单的事情，把值左移24个bit，也就是3个字节，所以系统留给了我们1个字节来表示内存的类型。下面是`VM_MAKE_TAG`的宏定义。
```c
#define VM_MAKE_TAG(tag) ((tag) << 24)
```
实际上苹果已经内置了很多默认的Type，下面列出一部分。
```c
#define VM_MEMORY_MALLOC 1
#define VM_MEMORY_MALLOC_SMALL 2
#define VM_MEMORY_MALLOC_LARGE 3
#define VM_MEMORY_MALLOC_HUGE 4
#define VM_MEMORY_SBRK 5// uninteresting -- no one should call
#define VM_MEMORY_REALLOC 6
#define VM_MEMORY_MALLOC_TINY 7
#define VM_MEMORY_MALLOC_LARGE_REUSABLE 8
#define VM_MEMORY_MALLOC_LARGE_REUSED 9
```
如果我们使用`VM_MEMORY_MALLOC_HUGE`来作为Type，再用VM Tracker观察会怎么样呢？下面是内存分配的代码。
```c
vm_address_t address;
vm_size_t size = 1024 * 1024 * 100;
vm_allocate((vm_map_t)mach_task_self(), &address, size, VM_MAKE_TAG(VM_MEMORY_MALLOC_HUGE) | VM_FLAGS_ANYWHERE);
```
下面是VM Tracker的截图。
![](https://user-gold-cdn.xitu.io/2018/1/23/16122167593fa63f?w=1240&h=498&f=jpeg&s=97696)

很明显VM Tracker认出了这块内存，并且将它的Type设定为MALLOC_HUGE。如果你想使用`vm_allocate`来分配和管理大内存，也可以设置一个Type，方便快速定位到自己的虚拟内存块。

# 总结
本文主要介绍了VM Tracker中关于虚拟内存的一些概念，以及如何自行分配虚拟内存。了解了这些之后，在分析内存暴涨或者泄漏时就有了新的思路，而不仅仅是局限于基于malloc内存块的内存分析了。