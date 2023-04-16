---
{"dg-publish":true,"permalink":"/Code/6.Go/Go.1c.Go并发原理/","title":"Go 并发原理","noteIcon":""}
---


# Go 并发原理

> [Golang的协程调度器原理及GMP设计思想](https://www.yuque.com/aceld/golang/srxd6d)

## 协程

协程是用户空间内与内核空间的系统线程相绑定的线程
协程与线程间的关系通过**协程调度器**控制

### 与线程的区别

线程由CPU调度，是**抢占式**的
线程可以理解为一个进程的执行实体，它是比进程力度更小的执行单元，也是真正运行在cpu上的执行单元，**线程是操作系统调度资源的基本单位**
一般OS线程栈大小为**2MB**

协程由用户态调度，通常情况下是**协作式**的，一个协程让出CPU后，才执行下一个协程(Go存在抢占式协程)
一个goroutine栈在其生命周期开始时占用空间很小（一般2KB），并且栈大小可以按需增大和缩小，goroutine的栈大小限制可以达到1GB，但是一般不会用到这么大

### Goroutine

通过 `go func()` 创建协程

Goroutine 父子协程之间不存在依赖关系，主协程例外，主协程退出时其他协程自动退出

同级协程之间的执行顺序随机

## 调度结构

>![image.png](https://image.jiang849725768.asia/2023/202303081555978.png)
> M(thread) G(goroutine) P(Processor)

```Go
type p struct {
    id          int32
    status      uint32 // one of pidle/prunning/...
  schedtick   uint32     // incremented on every scheduler call
    syscalltick uint32     // incremented on every system call
    m           muintptr   // back-link to associated m (nil if idle)
    // Queue of runnable goroutines. Accessed without lock.
    runqhead uint32
    runqtail uint32
    runq     [256]guintptr
    runnext guintptr
  ... 
}
```

**全局队列**：存放等待运行的G
**P的本地队列**：同样存放等待运行的G，数量不超过256个
执行过程中新建G'时，G'优先加入到P的本地队列，如果队列满了，则会把G‘和本地队列中前一半的G一起打乱顺序后移动到全局队列
**P列表**：所有的P都在程序启动时创建，并保存在数组中，最多有`GOMAXPROCS`个
**M**：线程想运行任务就得获取P，从P的本地队列获取G，P队列为空时，M会尝试从全局队列**拿**一批G放到P的本地队列，否则从其他P的本地队列**偷**一半放到自己P的本地队列。M运行G，G执行之后，M会从P获取下一个G，不断重复下去

**P的数量**由启动时环境变量`$GOMAXPROCS`或者是由`runtime`的方法`GOMAXPROCS()`决定。这意味着在程序执行的任意时刻都只有`$GOMAXPROCS`个goroutine在同时运行，`$GOMAXPROCS`通常等同于系统CPU核数
**M的数量**的最大值为10000，由runtime/debug中的SetMaxThreads函数设置，但实际上受系统限制通常无法达到
M与P的数量没有绝对关系，一个M阻塞，P就会去创建或者切换另一个M，所以即使P的默认数量是1，也有可能会创建很多个M出来

> ![image.png|325](https://image.jiang849725768.asia/2023/202303081640533.png)

**M0**是启动程序后的编号为0的主线程，这个M对应的实例会在全局变量runtime.m0中，不需要在heap上分配，M0负责执行初始化操作和启动第一个G， 之后M0和其他的M一样受P控制

**G0**是每次启动一个M时首先创建的goroutine，仅用于负责调度队列中的G
G0不指向任何可执行的函数。每个M都会有一个自己的G0，在调度或系统调用时会使用G0的栈空间，全局变量的G0是M0的G0

没有G可执行的M进入**自旋状态**，以在新G创建时能够立刻有M可以运行，系统中最多有`GOMAXPROCS`个自旋的线程，多余的线程进入**休眠状态**

## 调度时机

### 主动调度

协程可以选择主动让渡自己的执行权，这主要通过在代码中主动执行`runtime.Gosched()`函数实现
- 主动调度会从当前协程g切换到g0并更新协程状态由运行中`_Grunning`变为可运行`_Grunnable`
- 然后通过`dropg()`取消g与m的绑定关系
- 接着通过`globrunqput()`将g放入到全局运行队列中
- 最后调用`schedule()`函数开启新一轮的调度循环

### 被动调度

当某个G执行时发生了syscall或其余阻塞操作，M会阻塞，如果当前有其他G在本地队列中等待执行，runtime会把这个线程M与P分离(detach)，然后复用空闲线程(无则创建一个新线程)来服务于这个P
当M结束阻塞时会尝试连接一个空闲的P，之前的阻塞G进入到这个P的本地队列。如果获取不到P，那么这个M进入休眠状态， 加入到空闲线程中，然后这个G会被放入全局队列中
若G创建了G'且执行时进行了非阻塞系统调用，则M结束调用后优先尝试获取原P

当协程休眠、通道堵塞、网络堵塞、垃圾回收导致暂停时，协程会被动让渡出执行的权利给其他可运行的协程继续执行。调度器通过`gopark()`函数执行被动调度逻辑。`gopark()`函数最终调用`park_m()`函数来完成调度逻辑。

- 首先会从当前协程g切换到g0并更新协程状态由运行中`_Grunning`变为等待中`_Gwaiting`；
- 然后通过`dropg()`取消g与m的绑定关系；
- 接着执行`waitunlockf`函数，如果该函数返回`false`,则协程g立即恢复执行，否则等待唤醒；
- 最后调用`schedule()`函数开启新一轮的调度循环。

### 抢占调度

go应用程序在启动时会开启一个特殊的线程来执行系统监控任务，系统监控运行在一个独立的工作线程m上，该线程不用绑定逻辑处理器p。系统监控每隔10ms会检测是否有准备就绪的网络协程，并放置到全局队列中。

为了保证每个协程都有执行的机会，系统监控服务会对执行时间过长(`>10ms`)的协程、或者处于系统调用(`>20us`)的协程进行抢占。抢占的核心逻辑通过`retake()`函数实现
