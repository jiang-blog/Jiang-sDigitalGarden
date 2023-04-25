---
{"dg-publish":true,"permalink":"/Code/6.Go/Go.1c.Go并发/","title":"Go 并发","noteIcon":""}
---


# Go 并发

> [Golang的协程调度器原理及GMP设计思想](https://www.yuque.com/aceld/golang/srxd6d)

## 协程

协程是用户空间内与内核空间的系统线程相绑定的线程
协程与线程间的关系通过*协程调度器*控制

### 与线程的区别

线程由CPU调度，是**抢占式**的
线程可以理解为一个进程的执行实体，它是比进程力度更小的执行单元，也是真正运行在cpu上的执行单元，**线程是操作系统调度资源的基本单位**
一般OS线程栈大小为**2MB**

协程占用了更小的内存空间，也降低了上下文切换的开销
协程由用户态调度，通常情况下是**协作式**的，一个协程让出CPU后，才执行下一个协程(Go存在抢占式协程)
一个 goroutine 栈在其生命周期开始时占用空间很小（一般2KB），并且栈大小可以按需增大和缩小，goroutine 的栈大小限制为1GB，但是一般不会用到这么大

### Goroutine

通过 `go func()` 创建协程

Goroutine 父子协程之间不存在依赖关系，主协程例外，主协程退出时其他协程自动退出

同级协程之间的执行顺序随机

## 调度结构

 > [Go 语言调度器与 Goroutine 实现原理 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#652-%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84)

Go 调度器也叫 Go 运行时调度器，或 *Goroutine 调度器*，指的是由运行时在用户态提供的多个函数组成的一种机制，目的是为了高效地调度 G 到 M 上去执行

**M(thread) G(goroutine) P(Processor)**
>![image.png](https://image.jiang849725768.asia/2023/202303081555978.png)

**全局队列**：存放等待运行的 G
**P 的本地队列**：同样存放等待运行的 G，数量最大为256个
执行过程中新建G'时，G'优先加入到P的本地队列，如果队列满了，则会把G‘和本地队列中前一半的G一起打乱顺序后移动到全局队列

### G

Goroutine 是 Go 语言调度器中待执行的任务，在 Go 语言运行时使用私有结构体 `runtime.g` 表示

```Go
type g struct {
	stack       stack     // 当前 Goroutine 的栈内存范围 [stack.lo, stack.hi)
	stackguard0 uintptr   // 用于调度器抢占式调度
	preempt       bool    // 抢占信号
	preemptStop   bool    // 抢占时将状态修改成 `_Gpreempted`
	preemptShrink bool    // 在同步安全点收缩栈
	
	_panic       *_panic  // 最内侧的 panic 结构体
	_defer       *_defer  // 最内侧的延迟函数结构体

	m              *m     // 当前 Goroutine 占用的线程，可能为空
	sched          gobuf  // 存储 Goroutine 的调度相关的数据
	atomicstatus   uint32 // Goroutine 的状态
	goid           int64  // Goroutine 的 ID
  
	...
}
```

`stackguard0`字段被设置成 `StackPreempt` 意味着当前 Goroutine 发出了抢占请求
`goid` 对开发者不可见，Go 团队认为引入 ID 会让部分 Goroutine 变得更特殊，从而限制语言的并发能力

`sched` 字段的 `runtime.gobuf` 结构体构成如下

```go
type gobuf struct {
	sp   uintptr     // 栈指针
	pc   uintptr     // 程序计数器
	g    guintptr    // 持有该gobuf的goroutine
	ret  sys.Uintreg // 系统调用的返回值
	...
}
```
`gobuf` 中的内容会在调度器保存或者恢复上下文的时候用到，其中的栈指针和程序计数器会用来存储或者恢复寄存器中的值，改变程序即将执行的代码

#### 数量

Go 对 Goroutine 的数量没有限制，但 Goroutine 的实际数量受到系统资源的限制
可以使用协程池来控制协程的并发数量

#### 状态

除了几个已经不被使用的以及与 GC 相关的状态之外，Goroutine 可能处于以下 9 种状态：

|     状态      | 描述                                                                                        |
|:-------------:| ------------------------------------------------------------------------------------------- |
|   `_Gidle `   | 刚刚被分配并且还没有被初始化                                                                |
| `_Grunnable`  | 没有执行代码，没有栈的所有权，存储在运行队列中                                              |
|  `_Grunning`  | 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P                                 |
|  `_Gsyscall`  | 正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上   |
|  `_Gwaiting`  | 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上 |
|   `_Gdead`    | 没有被使用，没有执行代码，可能有分配的栈                                                    |
| `_Gcopystack` | 栈正在被拷贝，没有执行代码，不在运行队列上                                                  |
| `_Gpreempted` | 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒                              |
|   `_Gscan`    | GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在                                     |

> ![2020-02-05-15808864354615-golang-goroutine-state-transition.png (1201×430) (draveness.me)](https://img.draveness.me/2020-02-05-15808864354615-golang-goroutine-state-transition.png)

### M

M 是操作系统线程，使用私有结构体 `runtime.m` 表示

```go
type m struct {
	g0     *g       // 持有调度栈的 Goroutine
	curg   *g       // 在当前线程上运行的用户 Goroutine

	p      puintptr // 正在运行代码的处理器
	nextp  puintptr // 暂存的处理器
	oldp   puintptr // 执行系统调用之前使用线程的处理器
	...
}
```
`g0` 是一个运行时中比较特殊的 Goroutine，它会深度参与运行时的调度过程，包括 Goroutine 的创建、大内存分配和 CGO 函数的执行

**M0**是启动程序后的编号为0的主线程，这个 M 对应的实例会在全局变量 `runtime.M0`中，不需要在 heap 上分配，M0负责执行初始化操作和启动第一个 G，之后 M0和其他的 M 一样受 P 控制

**G0**是每次启动一个M时首先创建的goroutine，仅用于负责调度队列中的G
G0不指向任何可执行的函数。每个 M 都会有一个自己的 G0，在调度或系统调用时会使用 G0的栈空间
全局变量的 G0是 M0的 G0

#### 数量

调度器最多可以创建 10000 个线程 M，由 `runtime.debug` 中的 `SetMaxThreads` 函数设置，但实际上受系统限制通常无法达到
其中大多数的线程都不会执行用户代码(可能陷入系统调用)，最多只会有 `GOMAXPROCS` 个活跃线程能够正常运行

在大多数情况下，我们都会使用 Go 的默认设置，也就是线程数等于 CPU 数，默认的设置不会频繁触发操作系统的线程调度和上下文切换，所有的调度都会发生在用户态，由 Go 语言调度器触发，能够减少很多额外开销

### P

处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率

```Go
type p struct {
  id          int32
  status      uint32     // one of pidle/prunning/...
  schedtick   uint32     // incremented on every scheduler call
  syscalltick uint32     // incremented on every system call
  
  m           muintptr   // 反向存储的线程
  // 处理器持有的运行队列. Accessed without lock.
  runqhead uint32
  runqtail uint32
  runq     [256]guintptr
  runnext guintptr
  ... 
}
```

#### 数量

因为调度器在启动时就会创建 `GOMAXPROCS` 个处理器，所以处理器 P 的数量一定会等于 `GOMAXPROCS`，这些处理器会绑定到不同的内核线程上，`GOMAXPROCS` 通常等同于系统 CPU 核数，以最大程度地利用 CPU

M 与 P 的数量没有绝对关系，一个 M 阻塞，P 就会去创建或者切换另一个 M，所以即使 P 的默认数量是1，也有可能会创建很多个 M 出来

#### 状态

|    状态     | 描述                                                                               |
|:-----------:| ---------------------------------------------------------------------------------- |
|  `_Pidle`   | 处理器没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空 |
| `_Prunning` | 被线程 M 持有，并且正在执行用户代码或者调度器                                      |
| `_Psyscall` | 没有执行用户代码，当前线程陷入系统调用                                             |
| `_Pgcstop`  | 被线程 M 持有，当前处理器由于垃圾回收被停止                                        |
|  `_Pdead`   | 当前处理器已经不被使用                                                             |

### 流程

> ![image.png|325](https://image.jiang849725768.asia/2023/202303081640533.png)

没有 G 可执行的 M 进入**自旋状态**，以在新 G 创建时能够立刻有 M 可以运行，

#### 调度器启动

运行时通过 `runtime.schedinit` 初始化调度器
```go
func schedinit() {
	_g_ := getg()
	...

	sched.maxmcount = 10000 // 最大线程数

	...
	sched.lastpoll = uint64(nanotime())
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
}
```

获取环境变量 `GOMAXPROCS` 后就会调用 `runtime.Procresize` 更新程序中处理器的数量，在这时整个程序不会执行任何用户 Goroutine，调度器也会进入锁定状态，runtime.Procresize 的执行过程如下：

1. 如果全局变量 `allp` 切片中的处理器数量少于期望数量，会对切片进行扩容
2. 使用 new 创建新的处理器结构体并调用 `runtime.P.Init` 初始化刚刚扩容的处理器切片队列
3. 通过指针将线程 `M0` 和处理器 `allp[0]` 绑定到一起
4. 调用 `runtime.P.Destroy` 释放不再使用的处理器结构
5. 通过截断改变全局变量 allp 的长度保证与期望处理器数量相等
6. 将除 `allp[0]` 之外的处理器 P 全部设置成 `_Pidle` 并加入到全局的空闲队列中

调用 `runtime.procresize` 是调度器启动的最后一步，在这一步过后调度器会完成相应数量处理器的启动，等待用户创建运行新的 Goroutine 并为 Goroutine 调度处理器资源

#### 创建 G

某一绑定线程 M1的处理器 P1 上的协程 G1 通过 go func() 创建新 G2 后，由于局部性，G2优先放入 P 的本地队列

当创建的 G 超过 P1 的本地队列的长度(256)时，P1 本地队列中前半部分的协程(G2~G129)和 G258一起打乱顺序放入全局队列，本地队列中剩下的协程(G130~G257)往前移动

**创建新的 G 时，运行的 G 会尝试唤醒其他空闲的 M 去绑定 P 以执行任务**
如果 G2 唤醒了 M2，M2 成功与 P2绑定后，会先运行 M2 的 G0，这时 M2 没有从 P2 的本地队列中找到 G，会进入*自旋状态*
自旋状态的 M 会不断尝试从全局空闲线程队列里面获取 G 放到 P 本地队列去执行，系统中最多有 `GOMAXPROCS` 个自旋的线程，多余的线程进入*休眠状态*
获取的数量满足公式：`n = min(len(globrunqsize)/GOMAXPROCS + 1, len(localrunsize/2))`，保证每个 P 至少从全局队列中获取一个 G，同时不获取过多 G 以维持负载均衡


#### 切换 G

G1完成运行后 M1 上运行的 Goroutine 会切换为 G0 以负责调度协程的切换(运行 `schedule()` 函数)，从 M1 上 P1 的本地运行队列获取 G2 去执行

## 调度机制

### 主动调度

协程可以选择主动让渡自己的执行权，主要通过在代码中主动执行`runtime.Gosched()`函数实现
- 主动调度会从当前协程g切换到g0并更新协程状态由运行中`_Grunning`变为可运行`_Grunnable`
- 然后通过`dropg()`取消g与m的绑定关系
- 接着通过`globrunqput()`将g放入到全局运行队列中
- 最后调用`schedule()`函数开启新一轮的调度循环

### 任务窃取机制

处于自旋状态的 M 如果未从全局队列获取到 G ，为了提高并发执行的效率会从其他非绑定 P 的本地队列偷取一半的 G 来运行

### 交接机制

当某个 G 执行时发生了阻塞系统调用或其余阻塞操作将 M 阻塞时，如果当前有其他 G 在 P 的本地队列中等待执行，runtime 会把这个线程 M 与 P 分离，然后复用空闲线程(无则创建一个新线程)来服务于这个 P

当 M 结束阻塞时
1. 首先尝试获取原先的 P
2. 尝试连接一个空闲的 P，之前的阻塞 G 进入到这个 P 的本地队列
3. 如果获取不到 P，那么这个 M 进入休眠状态，加入到空闲线程中，与 M 绑定的 G 会被放入全局队列中

若G创建了G'且执行时进行了非阻塞系统调用，则M结束调用后优先尝试获取原P

当 G 因为休眠、通道堵塞、网络堵塞、垃圾回收导致暂停时，会被动让渡出执行的权利给其他可运行的协程继续执行
调度器通过 `gopark()` 函数执行被动调度逻辑。`gopark()` 函数最终调用 `park_m()` 函数来完成调度逻辑

1. 首先会从当前协程 G 切换到 G0，并更新 G 状态由运行中 `_Grunning` 变为等待中 `_Gwaiting`
2. 然后通过 `dropg()` 取消 g 与 m 的绑定关系
3. 接着执行 `waitunlockf` 函数，如果该函数返回 `false`，则协程 G 立即恢复执行，否则等待唤醒
4. 最后调用 `schedule()` 函数开启新一轮的调度循环

### 基于协作的抢占机制

Go 应用程序在启动时会开启一个特殊的线程来执行系统监控任务，系统监控运行在一个独立的工作线程 m 上，该线程不用绑定逻辑处理器 P
系统监控每隔10ms会检测是否有准备就绪的网络协程，并放置到全局队列中

为了保证每个协程都有执行的机会，系统监控服务会对执行时间过长(>10ms)的协程、或者处于系统调用(>20us)的协程进行抢占。抢占的核心逻辑通过 `retake()` 函数实现

### 基于信号的真抢占机制
