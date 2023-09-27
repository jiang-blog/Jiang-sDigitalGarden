---
{"dg-publish":true,"permalink":"/Code/4.OperatingSystem/OS.0.操作系统/","title":"操作系统","noteIcon":""}
---


# 操作系统

> ~~学习蒋炎岩《[操作系统](https://www.bilibili.com/video/BV12L4y1379V/?spm_id_from=333.788&vd_source=ea19bebc1add0fca9fa4aa2f334eb127)》课程的学习笔记
> 同时参考课程教科书《Operating Systems:Three Easy Pieces》的中文版 -- 《操作系统导论》~~

计算机的本质是状态机，一部分存储当前状态，一部分用于在每个时钟周期内计算并转移至下一状态。
程序也可以看成一个状态机，并且是计算机状态机的一部分，程序在计算机上的运行即为计算机从某一初始状态开始计算下一状态并转移(执行指令)。

为了能够多程序共用计算机中的硬件同时运行，产生了用于调度的操作系统。

## 系统内核

**内核是应用连接硬件设备的桥梁**，应用程序只需关心与内核交互，不用关心硬件的细节

> ![Kernel_Layout.png (1280×1011) (xiaolincoding.com)|600](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%86%85%E6%A0%B8/Kernel_Layout.png)

现代操作系统的内核一般会提供 4 个基本能力：
- **任务调度**，决定使用 CPU 的进程/线程
- **管理内存**，决定内存的分配和回收
- **硬件通信**，为进程与硬件设备之间提供通信能力
- **提供系统调用**，使应用程序通过系统调用运行需要更高权限的服务

大多数系统将内存分为两个区域：
- 内核空间，仅内核程序可访问，高权限
- 用户空间，专门给应用程序使用，低权限

应用程序通过系统调用进入内核空间执行程序

>![systemcall.png (1053×332) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%86%85%E6%A0%B8/systemcall.png)

应用程序使用系统调用时产生一个中断，CPU 中断当前在执行的用户程序，跳转到中断处理程序，也即内核程序开始执行，内核处理完后，主动触发中断，把 CPU 执行权限交回给用户程序，回到用户态继续工作

### Linux

特点：
- Multitask **多任务操作系统**
- SMP **对称多处理**
  -每个 CPU 地位相等，对资源的使用权限相同
  - 多个 CPU 共享同一个内存，每个 CPU 都可以访问完整的内存和硬件资源
- **可执行文件链接格式** ELF
  - Linux 操作系统中可执行文件的存储格式
  - ELF 把文件分段，每段都有自己的作用
- Monolithic Kernel **宏内核**
  - 系统内核的所有模块，比如进程调度、内存管理、文件系统、设备驱动等，都运行在内核态
  - 动态加载内核模块，例如大部分设备驱动是以可加载模块的形式存在的，与内核其他模块解耦

**微内核**：内核只保留最基本的能力，比如进程调度、虚拟机内存、中断等，把一些应用放到了用户空间，比如驱动程序、文件系统等
这样服务与服务之间是隔离的，单个服务出现故障或者完全攻击，也不会导致整个操作系统挂掉，提高了操作系统的稳定性和可靠性

**混合类型内核**：内核里面有一个最小版本的内核，其他模块在这个基础上搭建，实现时跟宏内核类似，把整个内核做成一个完整的程序，大部分服务都在内核中，像是以宏内核的方式包裹着一个微内核

### Windows

> ![windowNT.png (1024×1313) (xiaolincoding.com)|575](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%86%85%E6%A0%B8/windowNT.png)

特点：
- Multitask **多任务操作系统**
- SMP **对称多处理**
- **可移植执行文件** PE
- **混合型内核**
