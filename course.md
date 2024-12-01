---
Time: 2024.11.11
Instructor: 石磊
Title: 秋冬季训练营三阶段组件化内核设计与实践
---

三阶段是四阶段的准备，原则不淘汰。
课程中会出一些题目，会进行排名。
课后题有困难会发参考答案。

# 序章——组件化内核的意义与概念以及课程和实验的安排

###### 组件化内核目标

创建一种**基于组件**构造内核的方法，构造应对**不同场景**的**各种模式**内核。
基于的是同一个组件仓库

###### 内核组件化设计与传统设计思路的差异

1. **面向场景和应用需求构建内核**

组件化内核相当于一个“工厂”，组件相当于零件，生产过程主要是组装。

2. 以**统一视角看待各种模式、不同规模的内核**

复杂内核可以看作基于小的组件构造成的，由复杂性从低到高构造

###### 组件化内核的意义

组件化内核相对传统构建方式的优势：

1. 提高内核开发效率
    组件是良好封装的功能单元，直接通过接口调用。
    组件经过良好测试和实际验证，成熟稳定。
    在构建速度和质量两方面都能获得提升。
2. 降低内核维护难度
    内部功能之间的耦合降低
    组件隔离有利于缺陷隔离
    方便快速定位问题
3. 开展基于组件的功能复用和开发协作
    直接利用组件仓库中积累的组件
    团队内部分别开发

###### 组件化内核的概念

* 内核系统
  * 传统Kernel中，是一个承上启下的组件
  * 组件化内核要平替掉，是用内核组件构成的
* 内核组件
  * 用于构成内核系统偶的最基本单元，最小可部署单元。
  * 组件可以独立构建和分发，不能独立运行
  * 在Rust中，==[lib].crate

###### 主流内核模式特点对比

![image-20241111201027668](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111201027668.png)

###### 组件化内核实验思路

由组件增量的方式来构造Unikernel，当组件复杂的时候，可以通过扩展组件形成宏内核模式

![image-20241111201200517](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111201200517.png)

###### 组件化内核实验路线

###### 组件化内核实验列表

* U: `Unikernel`
* M: `宏内核`
* H: `Hypervisor`

| 实验名称 | 场景需求 | 涉及组件（增量） |
| --------------- | --------------- | --------------- |
| U.1 Hello | 输出信息到屏幕 | axruntime/axhal |
| U.2 Collections | 动态内存分配，支持Vec集合类型 | axalloc |
| U.3 ReadPFlash | 地址空间重映射，支持PFlash MMIO | pagetable |
| U.4 ChildTask | 主任务加载程序，子任务执行 | axtask/sched_fifo |
| U.5 MsgQueue | 父子任务基于队列通信，锁保护 | axsync |
| U.6 FairSched | 抢占式公平调度和WaitQueue | timer/sched_fair/waitq|
| U.7 ReadBlock | 从块设备加载数据，取代PFlash | axdriver/driver_virtio |
| U.8 LoadApp | 从Fat32文件系统加载应用和数据 | fat32/vfs |
| M.1 UserPrivilege | 以用户特权级运行应用程序 | axmm(uspace) |
| M.2 UserAspace | 管理多用户地址空间支持多应用 | --- |
| M.3 Process | 进程管理和Linux应用加载 | elf |
| H.1 VirtualMode | 切换进入虚拟化模式 | axvcpu |
| H.2 GuestAspace | 准备和启动虚拟机 | axmm(hy) |
| H.3 VmExit | 相应时钟中断、SBI调用、页面异常 | --- |

###### 建立实验环境

4. 测试环境是否正常

```bash
make pflash_img
make disk_img
make run
```

###### 第三阶段课程安排

* 第一周 组件化内核基础和Unikernel模式（以下4部分压缩合并到3次课）

1. 内核组件化基本概念、思路和框架 (U.1)
2. 内存管理 - 分页、地址空间(U.3)和动态内存分配(U.2)
3. 任务调度 - 任务与运行队列(U.4)、协作式调度(U.5) 、 抢占式调度(U.6)
4. 块设备驱动(U.7)和文件系统(U.8)

* 第二周 宏内核扩展（其中第3次课由郑友捷老师讲，内容待定）

5. 从Unikernel到宏内核的扩展 (M.1)
6. 用户地址空间管理和页面异常(M.2)
7. 进程和Linux应用支持(M.3)

* 第三周 Hypervisor扩展

8. 从Unikernel到宏内核的扩展 (H.1)
9. Guest地址空间管理(H.2)
10. VMExit各类情况处理与设备管理 (H.3)

![image-20241111201909696](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111201909696.png)

---

# 第一部分：Unikernel

![image-20241111201942338](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111201942338.png)

---

# U.1.0 Helloworld

![image-20241111202001716](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111202001716.png)

源码：`src/main.rs`
实验命令行：

```bash
make run A=tour/u_1_0
```

本节目标：

1. 从HelloWorld开始，建立Unikernel框架和核心组件。
2. 了解features在组件化内核构建中的作用

## 需要解决的问题

1. Unikernel模式内核的特点？

应用与内核：

1. 处于**同一特权级**——内核态
2. 共享**同一个地址空间**——相互可见，可以直接访问
3. 编译形成**一个Image**而后**一体运行**，Unikernel即是应用又是内核，是二者**合体**

优势：二者之间无隔离无切换，简单搞笑
劣势：二者之间无隔离无切换，安全性低

 一般来讲，会用虚拟化来增强安全
​ ==但是，难道这个过程不会增加性能消耗吗？==

Unikernel即是应用又是内核，在Hardware/Hypervisor之上，跑OS+App

2. Unikernel框架与构成框架的核心组件是如何构成的？

由裸机程序---》层次化---》组件化

HAL：硬件抽象层，低于这个抽象层就是体系结构相关的。

Runtime：非应用逻辑的通用功能，跟业务无关的抽象层

组件化抽象时，runtime层可以组件化

![image-20241111202625144](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111202625144.png)

![image-20241111202357533](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111202357533.png)

## 框架组件构成与协作过程

```
------引导 准备环境----》
axhal ---> axruntime ---> app: hello_world
^                                       |
|                                       |
+----------api:arceos---ulib:axstd------+
《--------运行 调用功能-------
```

### 引导过程：axhal(riscv64)

![image-20241111203336258](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111203336258.png)

### 引导过程：axruntime(1)

![image-20241111203654362](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111203654362.png)

### 引导过程：axruntime(2)

![image-20241111203748257](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111203748257.png)

### 引导过程：axruntime(3)

![image-20241111203903417](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111203903417.png)

### 引导完成转入运行阶段

api是统一的，但是ulib是跟语言相关的，具体实现在hal里面的，最终调用sbi::putchar(riscv的)

![image-20241111203921652](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111203921652.png)

## 日志级别控制与features

axfeat其实就是一个用于汇总与引用下部分组件的组件

![image-20241111204204123](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111204204123.png)

![image-20241111204438234](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111204438234.png)

## 小结

最基本的框架：核心组件axhal、axruntime、api、ulib以及上层应用

![image-20241111204648512](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111204648512.png)

## 课后练习

![image-20241111204724972](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111204724972.png)

---

# U.2.0 Collections

![image-20241111204826720](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111204826720.png)

源码：`src/main.rs`
实验命令行：

```bash
make run A=tour/u_2_0
```

本节目标：

1. 引入动态内存分配组件，支持Rust Collections类型
2. 动态内存分配的框架和算法

## 需要解决的问题

对基于“堆”的动态数据结构的支持？

Rust Collections标准类型需要动态内存分配支持

在Kernel开发层面，没有另外一个OS内核为其提供内存管理支持。

只能由内核自己实现global_allocator适配合

![image-20241111204953189](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111204953189.png)

## 内存分配

### 接口、框架与算法

![image-20241111205142150](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111205142150.png)

### 接口和数据结构

global_allocator就可以调用pageAllocator中的东西。

![image-20241111205400980](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111205400980.png)

### 框架初始化

![image-20241111205517168](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111205517168.png)

### 算法组件接口

![image-20241111205634765](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111205634765.png)

## 内存分配算法

### TLSF(Two-Level Segregated Fit) ArceUnikernel就是这个算法

![image-20241113210116682](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113210116682.png)
两级位图+一个列表

第一级位图中每一个位代表一个块

二级：大块中具体那些小块是空闲的，有几位就是对上面的块进行几等分。

### Buddy

LInux的页管理算法，在ArceOS作为字节分配器使用

![image-20241113210338502](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113210338502.png)

优先会找最小的，过于大就二分，直到找到合适的大小。

兄弟块会查找邻居，并进行合并

### Slab

Linux中用于做页的上一级——缓存管理。ArceOS也是做为字节分配器来管理

![image-20241113210602121](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113210602121.png)

# 课后练习1

HashMap中，需要体系结构的支持：随机数功能支持

参考官方的STD如何做的

![image-20241111205844042](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241111205844042.png)

---

# U.3.0 ReadPFlash

代码逻辑：从PFlash的地址开始，读出来前几个字节，按UTF-8打印出来。

==如果去掉paging features的时候会报错（地址异常）==

实验命令行：

```bash
make run A=tour/u_3_0
（尝试没有指定"paging"时的情况）
```

本节目标：

1. 引入页表管理组件，通过地址空间重映射，支持设备MMIO
2. 地址空间概念，重映射的意义，页表机制

## 需要解决的问题

1. PFlash的作用？
Qemu的PFlash模拟闪存磁盘，启动时自动从文件加载内容到固定的MMIO区域，而且对读操作不需要驱动，可以直接访问。

0号：被qemu保留作为拓展固件。

![image-20241113200343845](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113200343845.png)

2. 为何不指定"paging"时导致读PFlash失败？
ArceOS Unikernel包括两阶段地址空间映射，Boot阶段默认开启1G空间的恒等映射；
如果需要支持设备MMIO区间，通过指定一个feature-"paging"来实现重映射

```
axruntime: 分页二阶段，remap----------手动指定feature-"paging"
      ^
      | 引导过程
      |
axhal(boot): 分页一阶段，恒等映射-----默认开启分页机制
```

## 地址空间与启用

```
+-------------------------+                                                       +-----------------------+
|       unikernel         |                 分页启动前                            |      物理地址空间     |
|   +-----------------+   |------------------------------------------------------>|         Kernel        |
|   |     app         |   |                                                       |         sbi           |
|   +-----------------+   | 分页启动之后  +--------------+                        |                       |
|   +-----------------+   |-------------->| 虚拟地址空间 |----------------------->|       virtio slots    |
|   |     libos       |   |               +--------------+                        |           uart        |
|   +-----------------+   |                                                       |         plic          |
+-------------------------+                                                       +-----------------------+
          CPU                                   MMU                                          BUS
```

## 物理地址空间

![image-20241113200833529](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113200833529.png)

kernel开始的地址：0x8000_0000

## 分页--功能抽象和对应组件

![image-20241113201225614](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113201225614.png)

## 分页阶段

分页启动的两个阶段：早期启用（必须）和后期重建映射（可选）

### 1--早期启动（必须）



**恒等映射**：虚拟地址与物理地址一致

![image-20241113201510417](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113201510417.png)

**阶段1**: 内核启动的早期，采用规定的恒等映射的方式。但是只映射一部分物理空间。

```
                   物理空间                              虚拟空间
              +---------------+                       +-----------+
              |               |                       |           |
              |               | 0xffff_ffc0_c000_0000 |-----------|
              |               |                       |           |
              |               |                       |           |<----+
              |               | 0xffff_ffc0_8000_0000 |-----------|     |
              |               |                       |           |     |
              |               |                       |           |     |二步完成切换
              |               |                       |           |     |
0xC000_0000   |     Kernel    |---------------------->|           |     |
              |       |       |         第一步        |           |-----+
              |      sbi      |                       |           |
0x8000_0000   +---------------+---------------------->+-----------+
```

**目标**：完成Paging切换后，建立从虚拟空间`0xffff_ffc0_8000_0000 ~ 0xffff_ffc0_c000_0000`到物理空间`0x8000_0000 ~ 0xC000_0000`的映射，范围1G。

**两步完成Paging切换**：

1. 恒等映射保证虚拟空间与物理空间有一个相等范围的地址空间映射（0x8000_0000~0xC000_0000）。
    切换前后地址范围不变，但地址空间已经从物理空间切换到虚拟空间。
2. 给指令指针寄存器PC，栈寄存器sp等加偏移。
    如此在虚拟空间执行平移后，就完成到最终目标地址的映射。

![image-20241113201925626](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113201925626.png)

### 2--重建映射（可选）

![image-20241113202039702](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113202039702.png)

sbi由于不需要访问且无法访问，重映射之后，虚拟地址空间后就没有SBI了。
Kernel段的内容通过不同的权限限制被映射。（细粒度的映射）

**阶段 2**：指定paging feature的情况下，启动后期重建完整的空间映射。
注：paging不是决定分页是否启用，而是决定是否包含阶段2。

![image-20241113202225639](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113202225639.png)

意义：

1. 管理更大范围的地址空间
2. 分类和权限的粒度控制更加细致

## 小结

内存管理框架和功能：

1. 内存分配功能
    内含两类分配器，字节分配器和页分配器。
    框架与算法分离，松耦合支持多种内存分配算法。
2. 分页功能
    启动早期基于静态恒等映射完成分页切换，如果指定paging feature则会在启动后期重新建立范围更大，权限控制更细的映射。
3. ![image-20241113202421652](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113202421652.png)

---

# U.4.0 ChildTask

![image-20241113202604585](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113202604585.png)

![image-20241113202634381](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113202634381.png)

用新的线程来实现新的任务。

实验命令行：

```bash
make run A=tour/u_4_0
```

本节目标：

1. 启动第一个子任务，建立多任务的概念和基本框架
2. 框架要素：任务、运行队列

## 需要解决的问题

![image-20241113202744459](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113202744459.png)

## 任务和任务状态

![image-20241113203222327](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113203222327.png)

## 任务数据结构

is_idle, is_init是用来标记具有特殊性的任务

entry是任务逻辑入口

unikernel中的任务对等于线程（因为只有一个地址空间，unikernel没有进程，或者只有一个进程）

kstack：线程是有自己的栈空间

ctx：上下文类型TaskContext

tast_ext：扩展任务的属性

![image-20241113203439075](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113203439075.png)

## 通用调度框架

![image-20241113203742206](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113203742206.png)



## 接口

### 主要调度api

![image-20241113204006181](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113204006181.png)

## 调度框架初始化

![image-20241113204135358](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113204135358.png)

## 系统默认内置任务

![image-20241113204306992](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113204306992.png)

## 小结

![image-20241113204545776](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113204545776.png)

---

# U.5.0 MsgQueue

![image-20241113204651033](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113204651033.png)

实验命令行：

```bash
make run A=tour/u_5_0
```

## 需要解决的问题

![image-20241113204811391](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113204811391.png)

## 核心算法：context_switch

![image-20241113205150739](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113205150739.png)

## 协作式调度——CooperativeSched

![image-20241113205313217](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113205313217.png)

## 协作式调度算法FIFO机制

![image-20241113205733300](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113205733300.png)

## 协作式调度算法FIFO实现

![image-20241113205530073](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113205530073.png)

## 同步原语

![image-20241113205708966](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113205708966.png)

![image-20241113205904487](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113205904487.png)

---

上次的内存分配算法（见前面）



## 问答

内核在刚启动的时候，一定在高段地址上，因为是Kernel的代码。在paging之后，建立起映射。（没有映射的时候，是直接跑物理地址的，Paging之后才成了虚拟地址空间）

==这里有点疑惑==

内核为啥要映射到高位？：惯例，宏内核是从Unikernel扩展的，低段都是用户空间（惯例），比较方便

我们写的alloctor是怎么和vec的global_alloctor联系起来的？看一下上一次的PPT，当申请内存的时候（Vec），需要调用库中的global allocator。我们需要实现这个特征，可以理解为我们在实现后端。

# 课后练习2

![image-20241113210733136](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113210733136.png)

注意：我们需要检查两边会不会跨越边界，内存是否分配耗尽

![image-20241113210837442](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241113210837442.png)

接续上面：PageAllocator。

这个的背景：Bump的场景是有这样一个内核：在刚启动的时候，并不知道自己有多少物理内存

*对于Linux.会找到一个确定存在的小的内存的初始化，依照这个确定内存中，为早期的数据结构提供支持，运行一些算法，获得整个内存大小。*

---

# U.6.0 FairSched

![image-20241116173550346](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241116173550346.png)

实验命令行：

```bash
make run A=tour/u_6_0
make run A=tour/u_6_1
```

两个实验都能观察到：任务执行过程中被抢占

本节目标：

1. 抢占式调度机制，CFS调度策略
2. 时钟中断

## 抢占式调度——PreemptiveSched

**抢占式调度**：可以打断**当前任务**的执行，移交CPU执行权给当前“更”有资格的任务。抢占机制的根本保障是系统定时器。所以抢占针对的主要操作目标就是**current task当前任务**。

**机制与时机**：**不是无条件**的抢占，要两个条件都具备一是任务内部达到了某种条件，例如时间片耗尽；二是外部条件与时机，在preempt从disable到enable的那个状态切换点触发抢占。

![image-20241116180746499](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241116180746499.png)

## 抢占发生的条件与时机

![image-20241116181102064](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241116181102064.png)

1.   只有内外条件都满足时，才发生抢占；内部条件举例任务时间片耗尽，外部条件类似定义某种临界区，控制什么时候不能抢占，本质上它基于**当前任务的preempt_disable_count**。
2.   只在*禁用->启用* 切换的下边沿触发；下边沿通常在自旋锁解锁时产生，此时正是切换时机。
3.   推动内部条件变化和边沿触发产生的根本源是**时钟中断**。

## 外部条件

### 外部控制的抢占开关

抢占针对的目标就是当前任务，由外部控制的抢占开关是当前任务的**preempt_disable_count**
作为计数：0代表开抢占，大于0则关抢占。

![image-20241117113406047](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117113406047.png)

## 抢占式调度与协作式在框架上的关键区别

抢占式调度在协作式自主yield的基础上，引入**抢占式控制**和**时钟中断**，动态调整内外条件，触发优先级高的任务及时获得调度机会，避免个别任务长期不合理的占据CPU。

![image-20241117113705510](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117113705510.png)

## 时钟中断与抢占式调度

```
通过axhal注册时钟中断，定期触发axtask::on_timer_tick
				|
				|
				V
经由axtask::runqueue传递定时事件
				|
				|
				V
触发特定调度器的task_tick，决定是否标记抢占标志，并可能进一步的导致任务队列的重排
```

## 抢占式调度算法ROUND_ROBIN

### 机制

在协作式调度FIFO的基础上，由定时器定时递减**当前任务**的时间片，耗尽时允许调度，一旦外部条件符合，边沿触发抢占，当前任务排到队尾，如此完成各个任务的循环排列。核心目标是**当前任务**。

![image-20241117115136788](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117115136788.png)

### 实现

![image-20241117140917135](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117140917135.png)

## 抢占式调度算法CFS（Completely Fair Scheduler）

vruntime最小的任务就是优先权最高任务，即**当前任务**。计算公式：`vruntime = int_vruntime + (delta / weight(nic))`
系统初始化时，init_vruntime，delta，nice三者都是0

![image-20241117123507324](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117123507324.png)

![image-20241117140932256](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117140932256.png)

![image-20241117140942131](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117140942131.png)

## 小结

建立了基础的调度框架，引入三个系统任务辅助任务管理。目前可以基于最简单的协作式和抢占式调度算法，支持多任务。

![image-20241117143957905](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117143957905.png)

# U.7.0 ReadBlock

本节目标：

1.   从磁盘块设备读数据，替换PFlash
2.   发现设备相关驱动、块设备、VirtIO设备

实验命令行：

```
make run A=tour/u_7_0 BLK=y
```

![image-20241117144158794](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117144158794.png)

## 需要解决的问题

1.   设备管理框架
2.   设备发现与初始化
3.   中断机制与初始化

## 设备管理框架

**AllDevices**管理系统所有的设备，为上层的子系统如文件系统FS、网络协议栈NET提供访问服务。三种设备类型：

![image-20241117144511459](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117144511459.png)

## 设备发现与初始化过程

![image-20241117150954932](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117150954932.png)

## 基于总线发现设备——qemu平台示例

## virtio设备的probe过程

1.   qemu模拟器基于命令行产生设备
     ```bash
     $ bash-device virtio-blk-device, drive=disk0 -drive id=disk0, format=raw, file=disk.img
     ```

2.   qemu将设备mmio地址区域映射到Guest中qemu-virt平台默认有8个区域槽位，通常只有部分会形成映射，其他处于未映射状态，即表现为空设备

3.   virtio-mmio驱动逐个发请求区探查这些区域槽位
     对应映射设备相应请求，返回本设备的类型ID
     没有映射的槽位返回零，表示空设备

4.   virtio-mmio驱动把probe结果报告上呈

![image-20241117152643203](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117152643203.png)

## virtio驱动基本模型

**virtio驱动和virtio设备交互的两条路**

1.   主要基于vring环形队列：本质上是连续的Page页面，在Guest和Host都可见可写
2.   中断相应的通道：主要对等待读取大块数据时是有用的。

## 中断机制与初始化（以riscv64为例）

![image-20241117153509295](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241117153509295.png)

## 块设备驱动的组件构成

```
+---------------------------------------------------+
|					FileSystem						|
+---------------------------------------------------+
						|
						V
+---------------------------------------------------+
|				Trait: BlockDriverOps				|
+---------------------------------------------------+
		^				^				^
		|				|				|
+-----------+	+---------------+	+---------------+	
|	ramdisk	|	|	virtio-blk	|	| bcm2835sdhci	|
+-----------+	+---------------+	+---------------+
```

# U.8.0 LoadApp

本节目标：

1.   从文件系统加载应用和数据
2.   文件系统的初始化和文件操作

实验命令行：

```bash
make run A=tour/u_8_0 BLK=y
```

![image-20241118231106957](/home/marinatoo/.config/Typora/typora-user-images/image-20241118231106957.png)

## 文件系统

### 组件构成

*   应用：apps（fs（shell））
    *   应用shell本身模拟linux的shell，包含了一系列操作文件的子命令
*   框架：modules（axfs（fatfs））
    *   框架负责在启动时建立类似linux文件系统，在根目录下包含普通目录与文件，及dev目录
    *   axfs对应的是通用的目录和文件对象
*   基础设施具体类型：crates（axfs_vfs+axfs_ramfs+axfs_devfs）
    *   CFS定义文件系统的接口层
    *   具体的FS实现：ramfs和devfs

### 抽象与对应的数据结构

抽象对象：filesystem,dir和file

![image-20241118231834547](/home/marinatoo/.config/Typora/typora-user-images/image-20241118231834547.png)

### 节点的操作流程

第一步：获得Root目录节点

第二步：解析路径，逐级通过lookup方法找到对应节点，直至目标节点

第三步：对目标节点执行操作

```bash
VfsOps::root_dir()
		|
		V
VfsNodeOps::lookup()
		|
		V
VfsNodeOps::op_xxx()
```

### 实例——Ext2

![image-20241118232040912](/home/marinatoo/.config/Typora/typora-user-images/image-20241118232040912.png)

### mount——意义1

mount可以理解为文件系统在内存中的展开操作（unflatten）
把易于存储的扁平化形态转化为易于搜索遍历的立体化形态。

![image-20241118232237750](/home/marinatoo/.config/Typora/typora-user-images/image-20241118232237750.png)

### mount——意义2

把一棵目录树的“根” “嫁接”到另一棵树的某个节点，两棵树就形成了一棵树。
两棵目录树基于的文件系统可以相同也可以不同。
另外，被mount的结点及其子孙节点都会被遮蔽，直到unmount.
lookup操作到达mount点时，将会发生访问目录树的切换

![image-20241118232956868](/home/marinatoo/.config/Typora/typora-user-images/image-20241118232956868.png)

### 在mount点上lookup的特殊处理

![image-20241118233021279](/home/marinatoo/.config/Typora/typora-user-images/image-20241118233021279.png)

![image-20241118233027465](/home/marinatoo/.config/Typora/typora-user-images/image-20241118233027465.png)

---

# 第二部分 Monolithic Kernel宏内核

![image-20241120210805385](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241120210805385.png)

## 课程安排

主要内容：基于组件化方法构建宏内核模式

*   第一次：从Unikernel到宏内核
    *   以构建最小化的宏内核为目标，说明：
    *   宏内核的特点、与Unikernel的差异分析、框架和组件组成、具体实现示例
*   第二次：地址空间管理和支持Linux应用
    *   1.   用户地址空间映射、缺页加载机制；
        2.   支持运行最简单的Linux原始应用
*   第三次：宏内核的开发实践经验（by 郑友捷老师）
    *   目前宏内核方向的课程基础Starry，主要是由郑老师负责开发的，他也是第四阶段——宏内核方向的实习指导。

---

# M.1.0 UserPrivilege 跨越模式边界

本节目标：

1.   从Unikernel扩展为宏内核的基本条件：特权级、地址空间、系统调用等
2.   用户特权级和特权上下文切换机制

实验命令行：

```bash
make payload ./update_disk.sh ./payload/orgin/orgin
make run A=tour/m_1_0 BLK=y
```

![image-20241120211603942](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241120211603942.png)

## 宏内核模式相对Unikernel模式的特点

![image-20241120211812465](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241120211812465.png)

Unikernel模式的应用与内核：

1.   处于同一特权级——内核特权级
2.   共享同一地址空间——相互可见
3.   编译形成一个Image，一体运行
4.   Unikernel既是应用又是内核，二者合体

相对于Unikernel，宏内核的特点：

1.   增加一个权限较低的**用户特权级**来运行应用。
2.   为应用创建独立的**用户地址空间**，与内核隔离。
3.   内核运行时可以随时加载**应用Image**投入运行。
4.   应用与内核界限分明。

## 需要解决的问题

**如何以Unikernel为基础，构建最小化宏内核？**

![image-20241120212635968](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241120212635968.png)

分析从Unikernel基础到目标最小化宏内核需要完成的增量工作：

1.   用户地址空间的创建和区域映射
2.   在异常中断响应的基础上增加系统调用
3.   复用Unikernel原来的调度机制，针对宏内核扩展Task属性
4.   在内核与用户两个特权级之间的**切换机制**

## 结合本节示例m_1_0说明实现过程

示例m_1_0的执行逻辑：

1.   创建用户地址空间
2.   加载应用origin到地址空间
3.   在地址空间中建立用户栈
4.   伪造一个返回应用的环境上下文现场
5.   把伪造现场设置到新任务的内核栈上
6.   启动新任务执行sret指令返回到用户态，从应用origin的entry开始执行
7.   应用origin只包含一行代码，即执行系统调用sys_exit
8.   注册在异常中断向量表中的系统调用响应函数处理sys_exit，内核退出

从这个分析可见，宏内核模式为用户应用建立了两类上下文
用户应用进程在它们之间交替运行：

1.   任务上下文——用户态：正常执行应用逻辑，也称为进程上下文
2.   异常上下文——内核态：处理系统调用与异常*中断上下文与异常上下文类似，不做讨论*

![image-20241120215007913](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241120215007913.png)

以下从两种上下文角度，说明m_1_0最小化宏内核的实现过程

## 本节示例m_1_0的启动过程概览

略

## 用户空间

### 地址空间管理与复制原理

页表分为高低两个部分：高端作为内核空间，低端作为用户应用空间。
以初始的内核根页表为模板，为每个应用进程复制独立页表。内核空间共享，用户空间独立使用。

![image-20241120220257896](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241120220257896.png)

### 概念与具体实现实现对应关系

页表用数据结构PageTable实现；地址空间用数据结构AddrSpace实现。

![image-20241120221715591](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241120221715591.png)

## 用户应用的编译链接

用户应用构建方式：Rust工具链+Rust嵌入式汇编

示例：payload/origin

```rust
#[no_mangle]
unsafe extern "C" fn _start() ->! {
    core::arch::asm!(
    	"li a7, 93",
    	"ecall",
    	options(noreturn)
    )
}
```

>   93是Sys_exit编号通过ecall触发系统调用

首先编译origin生成ELF格式，然后被工具链转化为二进程BIN格式。

```bash
cargo build -p origin ---target riscv64gc-unknown-none-elf --release
rust-objcopy --binary-architecture=riscv64 --strip-all -O binary [origin_elf] [origin_bin]
```

BIN格式作为exercises/m_1_0使用的用户应用image。

## 把用户应用安装到根文件系统

通过命令行make disk_img已经创建磁盘设备disk.img，并建立文件系统（fat32）。
安装用户应用就是mount该磁盘设备文件到./mnt目录，然后更新应用程序image。

例如，安装应用origin的二进制image：

```bash
mkdir -p ./mnt
mount $(1) ./mnt
mkdir -p ./mnt/sbin
cp /tmp/origin.bin ./mnt/sbin
umount ./mnt
```

## 把应用加载到用户地址空间

第一步，从文件加载代码到内存缓冲区。
第二步，为用户地址空间代码区域建立映射，拷贝代码到被映射页面中。

![image-20241121125203209](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241121125203209.png)

## 创建用户任务

### 任务扩展属性

TaskInner的扩展：
在其中增加一个ext的结构体成员，包装与宏内核相关的各个成员。
对原始Unikernel是空架构。

![image-20241121131005430](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241121131005430.png)

![image-20241121131052954](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241121131052954.png)

### 任务属性扩展机制的目的

对于各类内核模式，调度子系统机制是基本一致的，调度仅关心Task中与调度相关的属性，不关心资源属性。
模式之间区别主要就在于资源属性不同。
Unikernel模式下，资源都是全局的，Task几乎不包含资源属性；
宏内核模式下，以进程为单位管理和隔离资源，Task表示进程时，其中就要包含属于自己的资源引用。

**目的：尽可能复用共性的调度子系统，又能兼容处理各种模式的个性部分——资源管理。**

![image-20241121191757828](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241121191757828.png)

## 进入用户态

### 让出CPU，Kick Off 用户任务

对于Riscv等多数体系结构来说，并不存在一个专门指令实现从内核态到用户态的切换。
解决方法：在内核态伪造一个异常上下文现场，假装来自用户态，然后用sret指令返回去。

![image-20241121191938535](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241121191938535.png)

### 示例：m_1_0对系统调用的响应

---

# M.2.0 UserAspace

本节目标：

1.   缺页加载的原理和机制
2.   内存映射相关的系统调用

实验命令行：

```bash
make payload
./update_disk.sh payload/origin/origin
make run A=tour/m_2_0 BLK=y
```

![image-20241122172108560](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241122172108560.png)

## 从实验m_1_0到m_2_0

按照如下方式，修改m_1_0关于初始化用户栈的代码，从直接映射改为Lazy映射。

```rust
- let ustack_top = init_user_stack(&mut uspace, true).unwrap();
+ let ustack_top = init_user_stack(&mut uspace, false).unwrap();
```

屏幕提示为处理的用户页异常：
![image-20241122172406664](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241122172406664.png)

参考用户应用origin的实现（汇编语言）：
“addi sp, sp, -4”,
“sw a0, (sp)”,
得知确实是第二行代码向栈的「基址地址-4」位置写入导致的缺页写异常。

对照m_2_0的代码，相对于m_1_0增加了缺页异常处理：
![image-20241122172715818](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241122172715818.png)

## 宏内核地址空间管理相关对象的层次构成

地址空间管理涉及的主要对象：AddrSpace，MemorySet，MemoryArea和Backend的两种实现。

AddrSpace：包含一系列有序的区域并对应一个页表。
MemorySet：对BTreeMap的简单封装，对空间下的各个MemoryArea进行有序管理。
MemoryArea：对应一个连续的虚拟地址内存区域，关联一个负责具体映射操作的后端Backend。
Backend：负责具体的映射操作，不同的区域MemoryArea可以应对不同的Backend。目前支持两种后端类型：Linear和Alloc。

![image-20241122174228482](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241122174228482.png)

## 地址空间区域MemorySet

![image-20241122174935994](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241122174935994.png)

## 地址空间区域映射后端Backend

后端负责针对空间中特定区域的具体的映射操作，Backend从实现角度是一个Trait。
![image-20241122175126858](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241122175126858.png)

## 缺页异常响应函数的实现

多级调用handle_page_fault，最终到backend::alloc中的实现。
![image-20241122180615053](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241122180615053.png)

见：`modules/axmm/src/backend/alloc.rs`

## 系统调用sys_mmap的实现

实现方法与缺页异常处理的流程类似，触发原因不同。sys_mmap由应用调用libc的某些方法触发。
![image-20241123152242201](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241123152242201.png)

见：`modules/axmm/src/backend/alloc.rs`

---

# M.3.0 LinuxApp

本节目标：

1.   glibc/musl-libc与内核的参数协同
2.   启动Linux原始应用hello

实验命令行：

```bash
make payload
./update_disk.sh payload/hello_c/hello
make run A=tour/m_3_0 BLK=y
```

![image-20241123153425005](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241123153425005.png)

## 需要解决的问题

如何让Linux的原始应用（二进制）直接在我们的宏内核上直接运行？

![image-20241123153852948](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241123153852948.png)

**在应用和内核交互界面上实现兼容**。
兼容界面包含三类：

1.   syscall
2.   procfs & sysfs等伪文件系统
3.   应用、编译器和libc对地址空间的假定，涉及某些参数定义或某些特殊地址的引用

## ELF格式应用的加载

多少应用在编译时都默认采取ELF格式，内核负责解析和加载各段到用户地址空间中的适当位置。示例m_3_0使用的应用hello的格式：![image-20241123154729966](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241123154729966.png)

需要获取的信息：

1.   Entry point应用的入口地址
2.   两个Type为LOAD的段，表示需要加载。分别是代码段和数据段，从Flag属性可以区分。
     注意：数据段的文件内偏移Offset和虚拟地址VirtAddr不一样，且FileSiz和MemSiz不一样。

需要注意文件内偏移和预定的虚拟内存空间内偏移可能不一致，特别是数据段部分。

![image-20241123155409192](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241123155409192.png)

通常ELF为了节省空间，紧凑存储，从Offset和FileSiz定位段。
内核根据VirtAddr和MemSiz把段安置到目标虚拟内存位置。
由于BSS部分全零，所以ELF文件中只是标志位置和长度，不存实际数据，内核直接预留空间后**清零**。

## 应用的用户栈初始化

Linux应用基于glibc/musl-libc等库编译，libc在调用应用的main之前，检查用户栈上的参数等内容。
而应用启动之后，也可能会调用这些参数。内核需要在切换到首应用前，为应用准备栈上内容。

![image-20241123165751121](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241123165751121.png)

## 支持系统调用的层次结构

宏内核的系统调用实现路径大致与Unikernel并列。

![image-20241123165859509](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241123165859509.png)

## 示例m_3_0涉及的系统调用

示例m_3_0基于musl工具链以静态方式编译，工具链为应用附加的部分也会调用syscall。

![image-20241123170011311](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241123170011311.png)

## 对Linux常用文件系统的支持

Linux常用的文件系统包括ProcFS，SysFS和DevFS，与普通文件系统不同，它们属于伪文件系统，具有相同的接口和抽象，但是Backend却不是普通的数据。

![image-20241123170139921](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241123170139921.png)

# 第三部分 组件化扩展——Hypervisor

## 虚拟化和Hypervisor的概念

基于一台物理计算机系统建立一台或者多台虚拟计算机系统（虚拟机），每台虚拟机拥有自己的独立的虚拟机硬件，支持一个独立的虚拟执行环境。每个虚拟机中运行的操作系统，认为自己独占该执行环境，无法区分环境是物理机还是虚拟机

![image-20241126113426244](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126113426244.png)

## Hypervisor与模拟器Emulator的区别

根本区别：虚拟运行环境和支撑它的物理运行环境的体系结构即ISA是否一致。
![image-20241126114902308](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126114902308.png)

根据1974年，Popek和Goldberg对虚拟机的定义：
虚拟机可以看作是物理机的一种高效隔离的复制，蕴含三层含义：同质、高效和资源受控。
同质要求ISA的同构，高校要求虚拟化消耗可忽略，资源受控要求中间层对物理资源的完全控制。
Hypervisor必须符合上述要求，而模拟器更侧重的是仿真效果，对性能效率通常没有硬性要求。

## 虚拟化类型

主要包括两种类型：区别在于构成的层级、引导的过程、实现的基础。

![image-20241126121605575](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126121605575.png)

我们目前的实验仅针对I型虚拟化。

## Hypervisor支撑的资源对象层次

VM：管理地址空间；同一整合下级各类资源
vCPU：计算资源虚拟化，VM中执行流
vMem：内存虚拟化，按照VM的物理空间布局
vDevice：设备虚拟化：包括直接映射和模拟
vUtilities：中断虚拟化、总线发现设备等

**Hypervisor模拟了完整的物理机环境，物理环境下的资源在虚拟环境下都有对应**

![image-20241126121931394](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126121931394.png)

## vCPU与CPU的关系

![image-20241126122030219](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126122030219.png)

## 虚拟内存与物理内存的关系

![image-20241126122132601](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126122132601.png)

**Hypervisor基于页表扩展等机制建立和维护二者之间的对应关系**

## 虚拟设备与物理设备的关系

类似于CPU的情况 ，虚拟设备与物理设备的对应关系也是两种。
![image-20241126122310395](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126122310395.png)

## Hypervisor

![image-20241126122330773](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126122330773.png)

# H.1.0 VirtualMode

本节目标：

1.   Hypervisor实现的基础
2.   虚拟化模式切换机制概述

实验命令行：

```
make payload
./update_disk.sh payload/skernel/skernel
make run A=tour/h_1_0/ BLK=y
```

![image-20241126122455979](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126122455979.png)

## h_1_0实验目标和需要解决的问题

实验h_1_0建立最简化的Hypervisor，用于分析其原理。
<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126122710568.png" alt="image-20241126122710568" style="zoom:33%;" />

最简Hypervisor执行流程：

1.   加载GuestOS内核Image到新建地址空间
2.   准备虚拟机环境，设置特殊上下文
3.   结合特殊上下文和指令sret切换到V模式，即VM-ENTRY
4.   OS内核只有一条指令，调用sbi-call的关机操作
5.   在虚拟机中，sbi-call超出V模式权限，导致VM-EXIT退出虚拟机，切换回Hypervisor
6.   Hypervisor响应VM-EXIT的函数检查退出原因和参数，进行处理，由于是请求关机，清理虚拟机后，退出

需要解决的问题：

1.   准备能够进入虚拟化模式的特殊上下文
2.   为响应VM-EXIT设置处理函数

## Riscv64在特权级模式的H扩展

![image-20241126123406651](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126123406651.png)

特权级从三个扩展到五个，新增了与Host平行的Guest域

*   原来的S增强了对虚拟化支持的特性后，称它为HS
*   M/HS/U形成Host域，用原来运行I型Hypervisor或者II型的HostOS，三个特权级的作用不变
*   VS/VU形成Guest域，用来运行GuestOS，这两个特权级分别对应内核态和用户态
*   HS是关键，作为联通真实世界和虚拟世界的通道。体系结构设计了双向变迁机制

## 体系结构H扩展后的寄存器

H扩展后，S模式发生明显变化：原有s[xxx]寄存器组作用不变，新增hs[xxx]和vs[xxx]
hs[xxx]寄存器组的作用：面向Guest进行路径控制，例如异常/中断委托等
vs[xxx]寄存器组的作用：直接操纵Guest域中的VS，为其准备或设置状态

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126124316869.png" alt="image-20241126124316869" style="zoom:25%;" />

## 为进入虚拟化模式准备的条件（1）

ISA寄存器misa第七位代表Hypervisor扩展的启用/禁止。
对这一为写入0，可以禁止H扩展。

![image-20241126125723505](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126125723505.png)

## 为进入虚拟化模式准备的条件（2）

进入V模式路径的控制：**hstatus**==第七位==SPV记录上一次进入HS特权级前的模式，1代表来自虚拟化模式。
执行sret时，根据SPV决定是返回用户态，还是返回虚拟化模式。

![image-20241126130047020](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126130047020.png)

## 为进入虚拟化模式准备的条件（3）

Hypervisor首次启动Guest的内核之前，伪造上下文作准备：
设置Guest的sstatus，让其初始特权级为Supervisor；
设置Guest的sepc为OS启动入口地址VM_ENTRY，VM_ENTRY值就是内核启动的入口地址，
对于Riscv64，其地址是0x8020_0000。

![image-20241126130334496](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126130334496.png)

## 从Host到Guest的切换run_guest

 每个vCPU维护一组上下文状态，分别对应Host和Guest.
从Hypervisor切断到虚拟机时，暂存Host中部分可能被破坏的状态；加载Guest状态；
然后执行sret完成切换。封装该过程的专门函数run_guest。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126143015338.png" alt="image-20241126143015338" style="zoom:33%;" />

## 因为VM-Exit从Guest到返回Host

执行上页从Host的到Guest的逆过程，具体机制包含在run_guest之中。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126143613815.png" alt="image-20241126143613815" style="zoom:25%;" />

**返回过程包含在run_guest过程中，具体实现机制下节课与上页一起介绍**

## VM-Exit原因

### 执行特权操作

在Guest虚拟环境下，不存在M模式，没有OpenSBI。因而sbi-call超出当前环境的处理能力，触发模式切换，回到Host模式的Hypervisor中处理。
<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126144119549.png" alt="image-20241126144119549" style="zoom:25%;" />

### 嵌套的#PF

Guest环境的物理地址区间尚未映射，导致二次缺页，切换进入Host环境。（下节课内容）
<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241126144240260.png" alt="image-20241126144240260" style="zoom:25%;" />

---

# Hypervisor 阶段的简化实验模型

1.   模拟器qemu视为物理环境，仅包含单核CPU
2.   Hypervisor和虚拟机仅在单任务环境下运行，无并发
3.   省略VM虚拟机这一级对象，只有虚拟地址空间（aspace）和VCpu（上下文状态）两个对象
     1.   虚拟地址空间负责定位虚拟机资源
     2.   VCpu负责运行状态的跟踪和切换

![image-20241128092049659](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128092049659.png)

# H.2.0 GuestAspace

本节目标：

1.   两阶段地址映射的原理和机制
2.   为Guest映射PFlash扩展内存示例

实验命令行：
```bash
make pflash_img
make disk_img（在disk.img不存在时执行）
make A=tour/u_3_0
./update_disk.sh tour/u_3_0/u_3_0_riscv64-qemu-virt.bin
make run A=tour/h_2_0/ BLK=y
```

## h_2_0 实验目标和需要解决的问题

通过实验h_2_0分析两段地址映射原理和具体实现。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128094445145.png" alt="image-20241128094445145" style="zoom:25%;" />

虚拟机运行的实验内核是第一周的u_3_0： 从pflash设备读出数据，验证开头部分

有两种处理方式：

1.   模拟模式——为虚拟机模拟一个pflash，以file1为后备文件。当Guest读该设备时，提供file1文件的内容。
2.   透传模式——直接把宿主物理机（即qemu）的pflash透传给虚拟机

优劣势：模拟模式可以为不同虚拟机提供不同的pflash内容，但效率低；透传模式效率高，但捆绑了设备

需要解决的问题：

1.   两种模式都涉及两阶段映射的问题
2.   对两种模式的具体处理方法

## Guest与Host的地址空间关系

Guest是指虚拟机所在的执行环境；Host指Hypervisor所处的执行环境。
Hypervisor负责基于HPA面向Guest映射GPA，基本寄存器是hgatp；
Guest认为看到的GPA是“实际”的物理空间，它基于satp映射内部的GVA虚拟空间。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128102516885.png" alt="image-20241128102516885" style="zoom:25%;" />

## 体系结构riscv64对Guest地址映射支持

RISC-V G Stage

*   vsatp: Virtual Supervisor Address Translation and Protection Register,用于第一阶段页表翻译
*   hgatp: Hypervisor Guest Address Translation and Protection Register,用于第二阶段页表翻译
*   GVA->(vsatp)->GPA->(hgatp)->HPA

hgatp相对于satp在根页表级扩展了2位，总数达到41位

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128103452971.png" alt="image-20241128103452971" style="zoom:25%;" />

## 虚拟机物理地址空间布局

为虚拟机呈现与所模拟的物理平台相同的物理地址空间布局。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128103548971.png" alt="image-20241128103548971" style="zoom:25%;" />

## 实验H_2_0

### 结构

按照简化实验模型，实验只有一个VM，所以忽略该对象。
Hypervisor的主逻辑包含三个部分：

1.   准备VM的资源：VM地址空间和单个vCPU
2.   切换进入Guest的代码
3.   相应VMExit各种原因的代码

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128104429567.png" alt="image-20241128104429567" style="zoom:25%;" />

### 实现流程

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128104645036.png" alt="image-20241128104645036" style="zoom:25%;" />

## VCPU的准备

VCPU的实现与体系结构紧密相关，但操作上类似。
在准备进入虚拟化模式之前，设置GuestOS内核启动入口地址和Guest页表地址。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128105343974.png" alt="image-20241128105343974" style="zoom:25%;" />

## 在Host与Guest环境之间切换的框架

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128105421297.png" alt="image-20241128105421297" style="zoom:25%;" />

框架达到的目的：无论是首次还是将来进入Guest，都经由同一个入口vcpu_run

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128110606896.png" alt="image-20241128110606896" style="zoom:25%;" />

## 虚拟模式切换的关键

### run_guest

run_guest负责进入Guest，定义在`modules/riscv_vcpu/src/guest.S`
详细信息参考注释

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128113231653.png" alt="image-20241128113231653" style="zoom:25%;" />

### guest_exit

由于run_guest进入Guest时设置stvec为guest_exit，所以退出Guest时进入此处。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128113827494.png" alt="image-20241128113827494" style="zoom:25%;" />

## 设备模拟和透传（默认）两种模式的实现

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128113904897.png" alt="image-20241128113904897" style="zoom:25%;" />

## hypervisor与宏内核对比

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241128113933955.png" alt="image-20241128113933955" style="zoom:25%;" />

---

# 两段地址映射的补充说明

*   在Hypervisor内部：VA->PA
*   来自Guest则可能经历的两阶段映射GVA->GPA->HPA

![image-20241130145930122](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130145930122.png)

---

# H.3.0 & H.4.0: VmDev

本节目标：

1.   虚拟时钟中断支持；虚拟机外设的支持。
2.   虚拟机中运行宏内核和通过透传pflash加载应用。

## H.3.0实验目标——虚拟机时钟中断

通过实验h_3_0学习虚拟机时钟中断的实现原理，配合实验的Guest内核是u_6_0.
u_6_0中的抢占式调度机制依赖时钟中断，它启用时钟定时器，不断的触发时钟中断。
由于当前该内核是在Guest虚拟机中运行，Hypervisor必须能够模拟物理机的中断能力。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130151615841.png" alt="image-20241130151615841" style="zoom:25%;" />

执行命令（h_3_0实验步骤）：
```
make A=tour/u_6_0/
./update_disk.sh tour/u_6_0/u_6_0_riscv64-qemu-virt.bin
make run A=tour/h_3_0 BLK=y LOG=info
```

执行结果：在worker执行序列中夹杂着下面的日志

`Set timer...` Guest内核设置定时器
`timer irq emulation` 时钟中断注入虚拟机

## 回顾实验 U_6_0 的内容

在 u_6_0实验中，支持了抢占式调度算法CFS，该机制必须依赖于时钟中断

![image-20241130152016522](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130152016522.png)

## 在虚拟机中运行u_6_0面临的问题

物理环境或者qemu模拟器中，时钟中断触发时，能够正常通过stvec寄存器找到异常中断向量表，然后进入事先注册的响应函数。但是在虚拟机环境下，宿主环境下的原始路径失效了。有两种解决方案：

1.   启用Riscv AIA机制，把特定的中断委托到虚拟机Guest环境下。要求平台支持，且比较复杂。
2.   通过中断注入的方式来实现。即本实验采取的方式。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130153342992.png" alt="image-20241130153342992" style="zoom:25%;" />

## 体系结构对中断注入的支持

注入机制的关键是寄存器hvip，指示在Guest环境中，哪些中断处于Pending状态。
手册第18.2.3. Hypervisor Interrupt Register描述了hvip寄存器的作用和格式
<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130153712625.png" alt="image-20241130153712625" style="zoom:25%;" />
可以分别对外部中断、时钟中断和IPI中断进行设置，1表示有中断待处理。

## 时钟中断设置和注入的实现

支持虚拟机时钟中断需要实现两部分的内容：

1.   响应虚拟机发出的SBI-Call功能调用SetTimer
2.   响应宿主机时钟中断导致的VM退出，注入到虚拟机内部

![image-20241130154051471](http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130154051471.png)

## 时钟中断设置和注入的实现

两种VM-EXIT的处理都在vmexit_handler中，实现在文件modules/riscv_vcpu/src/vcpu.rs

## H.4.0 实验目标——支持宏内核作为GuestOS

h_4_0是虚拟机中启动宏内核的实验，配合实验的Guest内核是m_1_1。
m_1_1从pflash载入第一个用户应用并启动，响应syscall后退出。
Hypervisor透传pflash给虚拟机，让m_1_1内核能够正常完成工作。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130155154339.png" alt="image-20241130155154339" style="zoom:25%;" />

h_4_0实验步骤：

```bash
make A=tour/m_1_1
./update_disk.sh ./tour/m_1_1/m_1_1_riscv64-qemu-virt.bin
make run A=tour/h_4_0 BLK=y
```

略

## 虚拟机中支持宏内核要解决的问题

1.   Hypervisor需要加载宏内核的Image文件
2.   当宏内核启动后，Hypervisor支持某种块存储，让宏内核能够从中加载应用的Image文件前者直接从disk.img中加载，后者则参照h_2_0实验的方式，通过透传pflash，让宏内核能够从中读出应用Image，加载并执行。

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130155916924.png" alt="image-20241130155916924" style="zoom:25%;" />

## 虚拟机对设备的管理

管理上的层次结构：虚拟机（VM），设备组VmDevGroup以及设备VmDev。
Riscv架构下，虚拟机包含的各种外设通常是通过MMIO方式访问，因此主要用地址范围标记它们的资源。实现文件**tour/h_4_0/src/vmdev.rs**

<img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130160131527.png" alt="image-20241130160131527" style="zoom:25%;" />

## 实验H_4_0的实现

相对于h_2_0，当前实验支持了以宏内核为GuestOS内核，然后建立了设备管理机制。

1.   采取注册的方式，把各种支持设备注册到设备管理组中。
     <img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130160335287.png" alt="image-20241130160335287" style="zoom: 50%;" />

2.   发生页面异常时，基于地址查找目标设备，调用该设备的响应方法处理请求。
     <img src="http://marina-too.oss-cn-hangzhou.aliyuncs.com/img/image-20241130160446960.png" alt="image-20241130160446960" style="zoom:50%;" />