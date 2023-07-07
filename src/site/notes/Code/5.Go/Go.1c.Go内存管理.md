---
{"dg-publish":true,"permalink":"/Code/5.Go/Go.1c.Go内存管理/","title":"Go内存管理","noteIcon":""}
---


# Go内存管理

## 内存管理设计

### 分配方法

> [Go 语言内存分配器的实现原理 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/)

编程语言的内存分配器一般包含两种分配方法，一种是线性分配器，另一种是空闲链表分配器

#### 线性分配

线性分配是一种高效的内存分配方法，但是有较大的局限性
当我们使用线性分配器时，只需要在内存中维护一个指向内存特定位置的指针，如果用户程序向分配器申请内存，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置

虽然线性分配器实现为它带来了较快的执行速度以及较低的实现复杂度，但是线性分配器无法在内存被释放时重用内存

因为线性分配器具有上述特性，所以需要与合适的垃圾回收算法配合使用，例如：标记压缩（Mark-Compact）、复制回收（Copying GC）和分代回收（Generational GC）等算法，它们可以通过拷贝的方式整理存活对象的碎片，将空闲内存定期合并，这样就能利用线性分配器的效率提升内存分配器的性能

#### 空闲链表分配

空闲链表分配器可以重用已经被释放的内存，它在内部会维护一个类似链表的数据结构
当用户程序申请内存时，空闲链表分配器会依次遍历空闲的内存块，找到足够大的内存，然后申请新的资源并修改链表

因为不同的内存块通过指针构成了链表，所以使用这种方式的分配器可以重新利用回收的资源，但是因为分配内存时需要遍历链表，所以它的时间复杂度是 O(n)

空闲链表分配器可以选择不同的策略在链表中的内存块中进行选择，最常见的是以下四种：
- 首次适应 - 从链表头开始遍历，选择第一个大小大于申请内存的内存块
- 循环首次适应 - 从上次遍历的结束位置开始遍历，选择第一个大小大于申请内存的内存块
- 最优适应 - 从链表头遍历整个链表，选择最合适的内存块
- 隔离适应 - 将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块

### TCMalloc

TCMalloc(Thread Cache Malloc)核心理念是使用多级缓存将对象根据大小分类，并按照类别实施不同的分配策略

1. 每个 Thread 预分配有一块缓存 ThreadCache，每个 Thread 在申请内存时首先会先从 ThreadCache 申请，此外所有 ThreadCache 缓存共享一个叫 CentralCache 的中心缓存
2. 当 ThreadCache 的缓存不足时，就会从 CentralCache 获取，当 ThreadCache 的缓存充足或者过多时，则会将内存退还给 CentralCache
3. 当 CentralCache 没有足够内存时或 Thread 需要申请的大对象超过 Cache 容纳的内存块单元大小时，会从一个全局共享内存堆 PageHeap 取内存，当 CentralCache 内存过多或者充足，则将低命中内存块退还给 PageHeap

CentralCache 和 PageHeap 由于共享，访问需要加锁，ThreadCache 作为线程独立的第一交互内存，访问无需加锁

> ![image.png](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133032054-ea888b96-0fb0-46ea-ac26-4c38abc2b66f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_89%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fresize%2Cw_1038%2Climit_0)

TCMalloc 将虚拟内存空间划分为多份同等大小的**Page**，每个 Page 默认是8KB

多个连续的 Page 称之为一个**Span**，每个 Span 记录了第一个起始 Page 的编号 Start，和一共有多少个连续 Page 的数量 Length，Span 集合是以双向链表的形式构建，**TCMalloc 以 Span 为单位向操作系统申请内存**

在256KB 以内的小对象，TCMalloc 会将这些小对象集合根据 Page Size 划分成多个内存刻度，同属于一个刻度类别下的内存集合称之为属于一个**Size Class**
| **对象** | **容量**     |
| -------- | ------------ |
| 小对象   | (0,256KB/32Page]    |
| 中对象   | (256KB, 1MB/128Page] |
| 大对象   | (1MB, +∞)    |

>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133255704-7c07cb59-d879-468f-a925-d3494454cb7d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_82%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)
>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133299709-8c33bad3-a31f-4844-b07c-ad54a0dc64d4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_67%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

#### 对象分配

TCMalloc 的小对象分配
>![|525](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133672724-0ac13b26-1623-444a-8c81-0b2120b2e2fa.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_80%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

中对象分配
>![|525](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133803693-486e9b4a-ffb1-4932-a989-1df013b601c1.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_71%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

大对象分配
>![|525](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133987470-28f3feb2-8a9e-45be-a41b-596b1bd54e8d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_81%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

## Go 堆内存管理

> [9、一站式Golang内存管理洗髓经 (yuque.com)](https://www.yuque.com/aceld/golang/qzyivn#VyQ8I)
> [Go 语言内存分配器的实现原理 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/)

Go 内存管理借鉴了 TCMalloc 的设计思想，运行时根据对象的大小将对象分成微对象、小对象和大对象三种：
| 类别 | 大小      |
| ------ | ----------- |
| 微对象 | (0, 16B)    |
| 小对象 | [16B, 32KB] |
| 大对象 | (32KB, +∞) |

同时 GO 内存管理也使用 MCache，MCentral 和 MHeap 三个组件分级管理内存，最小的内存管理单元为 MSpan

>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134285363-999c7495-7834-4785-a6ea-c44b4615ff19.png)

### 内存单位

各层级间内存交换的单位如下：
>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134740690-c1fc15a5-af2a-474c-adaa-ddc4bb05a5e3.png)

#### Object

*Object*是 Go 内存管理内部用来存储对象内存的基本单元

*Object Size*指协程一次向 Golang 内存申请的 Object 大小

#### Page

*Page*的大小是8KB，是 Golang 内存管理系统与操作系统交互衡量内存容量的基本单元

#### MSpan

Span 的名称改为*mspan*，与 TCMalloc 一样，mspan 包含一组连续的 Page

**Size Class**在 Go 内存管理中针对 Object Size 来划分内存，Go 给内存池中固定划分了67种 Size Class，每一个 Size Class 都会存储特定大小的 Objects 并且包含特定数量的 Pages 以及 Objects，最大 Class 为 32KB

此外运行时中还包含 ID 为 0 的特殊 Size Class，它能够管理大于 32KB 的特殊对象

每个 Size Class **对应存在两个 Span Class**，一个用于存放需要 GC 扫描的对象(包含指针的对象)，另一个存放不需要 GC 扫描(不包含指针)的对象

每个 mspan 根据相应的 Object Size 分为 n(>=1)个 Object

| class | bytes/obj | bytes/span | objects | tail waste | max waste |
|:-----:|:---------:|:----------:|:-------:|:----------:|:---------:|
|   1   |     8     |    8192    |  1024   |     0      |  87.50%   |
|   2   |    16     |    8192    |   512   |     0      |  43.75%   |
|   3   |    24     |    8192    |   341   |     0      |  29.24%   |
|   4   |    32     |    8192    |   256   |     0      |  46.88%   |
|   5   |    48     |    8192    |   170   |     32     |  31.52%   |
|   6   |    64     |    8192    |   128   |     0      |  23.44%   |
|   7   |    80     |    8192    |   102   |     32     |  19.07%   |
|   …   |     …     |     …      |    …    |     …      |     …     |
|  67   |   32768   |   32768    |    1    |     0      |  12.50%   |

```go
type mspan struct {
  // 链表
	next *mspan
	prev *mspan

  // 页管理
  startAddr uintptr  // 起始地址
	npages    uintptr  // 页数
	freeindex uintptr  // 扫描页中空闲对象的初始索引

	allocBits  *gcBits // 标记内存的占用情况
	gcmarkBits *gcBits // 标记内存的回收情况
	allocCache uint64  // allocBits的补码，可以用于快速查找内存中未被使用的内存

  spanclass spanClass // 决定该mspan中存储的object大小和个数
	...
}
```
`spanClass` 是一个 uint8 类型的整数，它的前 7 位存储着跨度类的 ID，最后一位表示是否包含指针
运行时会使用 `runtime.mSpanList` 存储双向链表的头结点和尾节点并在线程缓存以及中心缓存中使用

当用户程序或者线程向 mspan 申请内存时，mspan使用 `allocCache` 字段以对象为单位在管理的内存中快速查找待分配的空间
如果能在内存中找到空闲的内存单元则直接返回，当内存中不包含空闲的内存时，上一级的组件 Mcache 会为调用 `runtime.mcache.refill` 更新内存管理单元以满足为更多对象分配内存的需求

### MCache

**MCache**对应 TCMalloc 的 ThreadCache，是 Go 中的线程缓存，Go 协程调度模型 GPM 中的每个 P 都有自己独立的一个 mcache，因此**协程逻辑层从 MCache 上获取内存不需要加锁**

每一个 mcache 中对应每个 Size Class 存在两个 mspan，一个用于存放需要 GC 扫描的对象(包含指针的对象)，另一个存放不需要 GC 扫描(不包含指针)的对象
因此每一个 mcache 都持有 68 * 2 个 mspan
每个 mspan 又根据相应的 Object Size 分为多个 object
这些 mspan 都存储在结构体的 `alloc` 字段中

MCache 在刚刚被初始化时不包含 mspan 实例，所有 mspan 都为空的占位符 `emptymspan`，当用户程序申请内存时 MCache 从上一级组件 MHeap 中获取新的 mspan 满足内存分配的需求

>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134640677-2a153c96-b7e8-46bc-86f3-dfaf50087329.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_84%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)
> 该图 Span Class 个数有误

对于 Span Class 为0和1(对应 Size Class 为0，Object Size 为0)的规格刻度内存，MCache 实际上没有分配任何内存
Go 内存管理对内存为0的数据申请特殊处理，直接返回一个固定内存地址**zerobase**

> 在 Go 中如\[0\]int、 struct{}所需要的空间大小大小均是0，这也是为什么很多开发者在通过 Channel 做同步时，发送一个 struct{}数据，因为不会申请任何内存，能够适当节省一部分内存空间

#### Tiny空间

每个 mcache 中包含一个特殊的**Tiny 空间**，其从 Object Size 为16B 的 mspan 中获取一个 object 作为 tiny 对象的分配空间，只有当该 object 中的所有对象都需要被回收时，整片内存才可能被回收

下面的这三个字段组成了微对象分配器，专门管理 16 字节以下的对象
```go
type mcache struct {
	tiny             uintptr
	tinyoffset       uintptr
	local_tinyallocs uintptr
}
```
**微分配器只会用于分配非指针类型的内存**，上述三个字段中 `tiny` 会指向堆中的一片内存，`tinyOffset` 是下一个空闲内存所在的偏移量，最后的 `local_tinyallocs` 会记录内存分配器中分配的对象个数

### MCentral

MCache 中某 mspan 空缺或用尽时，会向 MCentral 申请对应的 mspan，向 MCentral 申请 mspan 需要加互斥锁

MCentral 具体分为多个小 mcentral，每个 mcentral 都会管理某个 Size Class 的 mspan，它会同时持有两个 mspan 的集合，分别存储包含空闲对象和不包含空闲对象的内存管理单元，每个集合又分别存放已经清扫完的 spanSet 和未清扫完的spanSet
```go
type mcentral struct {
	spanclass spanClass
	partial  [2]spanSet //还未分配完的mspan
	full     [2]spanSet //没有空闲空间的mspan
}
```

Mcache 会通过 Mcentral 的 `runtime.mcentral.cacheSpan` 方法获取新的 mspan，该方法的实现比较复杂，可以将其分成以下几个部分：
1. 调用 `runtime.mcentral.partialSwept` 从清理过的、包含空闲空间的 `spanSet` 结构中查找可以使用的内存管理单元
2. 调用 `runtime.mcentral.partialUnswept` 从未被清理过的、有空闲对象的 `spanSet` 结构中查找可以使用的内存管理单元
3. 调用 `runtime.mcentral.fullUnswept` 获取未被清理的、不包含空闲空间的 `spanSet` 中获取内存管理单元并通过 `runtime.mspan.sweep` 清理它的内存空间
4. 调用 `runtime.mcentral.grow` 从堆中申请新的内存管理单元
5. 更新 mspan 的 `allocBits` 和 `allocCache` 字段，帮助快速分配内存

MCentral 中的 Span 不够时，会采用 `runtime.mcentral.grow` 扩容方法根据预先计算的 `class_to_allocnpages` 和 `class_to_size` 获取待分配的Pages以及 Size Class并调用 `runtime.mheap.alloc` 获取新的 mspan

### MHeap

MCentral中的Span不够时会向MHeap申请，同样需要加锁

MHeap 是内存分配的核心结构体，是堆内存的抽象，把从系统申请出的内存页组织成 mspan，并保存起来

MHeap以Page为内存单元进行管理，用来详细管理每一系列Page的结构称之为一个**HeapArena**，一个HeapArena占用内存64MB，里面的内存以page方式存放，所有的HeapArena组成的集合是一个**Arenas**

### 对象分配流程

Golang将对象分为tiny对象，小对象及大对象

**大量的微小对象可能会使 Object Size = 8B 的 mspan 产生许多空间浪费**，所以 Golang 内存管理决定将小于16B 的内存申请统一归类为 Tiny 对象申请
> ![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135299397-38b04b81-3179-44e4-81ca-bb476b69f22f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_73%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

对于**16B 至32KB**的内存申请，Go 采用小对象的分配流程

>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135402859-c6404c6a-f0fd-4bb9-bbef-0c5286b0b2a4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_100%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

对于大于32KB 的大对象申请直接从 MHeap 中分配，Proceesor 直接向 MHeap 申请对象所需要的适当 Pages

>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135463380-9ee93382-7deb-48c0-ab38-519d679101e4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_90%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

### 虚拟内存布局

>[Go 语言内存分配器的实现原理 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/#711-%E8%AE%BE%E8%AE%A1%E5%8E%9F%E7%90%86)

#### 线性内存

> ![heap-before-go-1-10.png (1207×330) (draveness.me)](https://img.draveness.me/2020-10-19-16031147347484/heap-before-go-1-10.png)

V1.10 以前 Go 在启动时会初始化整片虚拟内存区域，为 `spans`、`bitmap` 和 `arena` 三个区域分别预留了 512MB、16GB 以及 512GB 的内存空间
- `spans` 区域存储了指向内存管理单元mspan的指针，每个内存单元会管理几页的内存空间，每页大小为 8KB
- `bitmap` 用于标识 `arena` 区域中保存了对象的地址，bitmap中的每个字节都会表示堆区中的 32 字节是否空闲
- `arena` 区域是真正的堆区，运行时会将 8KB 看做一页，这些内存页中存储了所有在堆上初始化的对象

#### 稀疏内存

V1.11运行时使用二维的 `runtime.heapArena` 数组管理所有的内存，数组中每个元素管理 64MB 的内存空间

```go
type heapArena struct {
	bitmap       [heapArenaBitmapBytes]byte
	spans        [pagesPerArena]*mspan
	pageInUse    [pagesPerArena / 8]uint8
	pageMarks    [pagesPerArena / 8]uint8
	pageSpecials [pagesPerArena / 8]uint8
	checkmarks   *checkmarksMap
	zeroedBase   uintptr
}
```

`heapArena` 结构体中的 `bitmap` 和 `spans` 与线性内存中的 `bitmap` 和 `spans` 作用相同
`zeroedBase` 字段指向了该结构体管理的内存的基地址

上述设计将原有的连续大内存切分成稀疏的小内存，而用于管理这些内存的元信息也被切成了小块

> 不同平台和架构的二维数组大小可能完全不同，如果我们的 Go 语言服务在 Linux 的 x86-64 架构上运行，二维数组的一维大小会是 1，而二维大小是 4,194,304，因为每一个指针占用 8 字节的内存空间，所以元信息的总大小为 32MB。由于每个`heapArena`会管理 64MB 的内存，整个堆区最多可以管理 256TB 的内存，这比之前的 512GB 多好几个数量级

### 地址空间

所有的内存最终都是要从操作系统中申请的，所以 Go 语言的运行时构建了操作系统的内存管理抽象层，该抽象层将运行时管理的地址空间分成以下四种状态：
|    状态    | 解释                                                                                    |
|:----------:| --------------------------------------------------------------------------------------- |
|   `None`   | 内存没有被保留或者映射，是地址空间的默认状态                                            |
| `Reserved` | 运行时持有该地址空间，但是访问该内存会导致错误                                          |
| `Prepared` | 内存被保留，一般没有对应的物理内存访问该片内存的行为是未定义的可以快速转换到 Ready 状态 |
|  `Ready`   | 可以被安全访问                                                                          |

## Go 栈内存管理

> [Go 语言的栈内存和逃逸分析 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/)

Go 在栈内存管理中采用**连续栈**，其核心原理是每当程序的栈空间不足时，初始化一片更大的栈空间并将原栈中的所有值都迁移到新栈中，新的局部变量或者函数调用就有充足的内存空间

**可以认为 Go 语言的栈内存都是分配在堆上的**

如果运行时只使用全局变量来分配内存的话，势必会造成线程之间的锁竞争进而影响程序的执行效率，由于栈内存与线程关系比较密切，所以在每一个`mcache` 中都加入了栈缓存`stackcache`减少锁竞争影响

根据线程缓存和申请栈的大小，栈空间通过三种不同的方法分配：
- 申请栈空间小于32KB
  1. 先从`mcahce`中的栈缓存`stackcache`中分配
  2. 如果`stackcache`内存不足，则从全局栈缓存`stackpool`中分配
  3. 如果`stackpool`内存不足，则从逻辑处理器结构`p`中的`p.pagecache`中分配
  4. 如果`p.pagecache`内存不足，则从堆`mheap`中分配
- 申请栈空间大于32KB
  1. 直接从全局栈内存缓存池`stackLarge`中分配
  2. 全局栈内存缓存池`stackLarge`不足，则从逻辑处理器结构`p`中的`p.pagecache`中分配
  3. 如果`p.pagecache`内存不足，则从堆`mheap`分配

程序会在几乎所有的函数调用之前检查当前 Goroutine 的栈内存是否充足
使用连续栈机制时，栈空间不足导致的扩容会经历以下几个步骤：
1. 在内存空间中分配更大的栈内存空间
2. 将旧栈中的所有内容复制到新栈中
3. 将指向旧栈对应变量的指针重新指向新栈
4. 销毁并回收旧栈的内存空间

如果要触发栈的缩容，新栈的大小会是原始栈的一半，不过如果新栈的大小低于程序的最低限制 2KB，那么缩容的过程就会停止

## 内存泄漏

内存泄漏就是程序生命周期中一些对象不能被及时回收，一直占用着内存，导致这部分内存不可用的情况

Go语言中仍可能发生内存泄漏：预期的能很快被释放的内存由于附着在了长期存活的内存上、或生命期意外地被延长，导致预计能够立即回收的内存长时间得不到回收。

1. 预期能被快速释放的内存因被根对象引用而没有得到迅速的释放
e.g. 当有一个全局对象时，可能不经意间将某个变量附着在其上，且忽略释放该变量，则其内存永远不会得到释放
2. Goroutine泄漏
Goroutine在运行过程中消耗的用于维护上下文信息的内存不会被释放，当一个程序持续不断地产生新的Goroutine、且不结束已创建的Goroutine并复用这部分内存，就会造成内存泄露
3. Channel 泄漏
如果一个 goroutine 阻塞在 channel 处而 channel 数据一直未改变，则该 goroutine 会被动永久休眠，整个 goroutine 及其执行栈都得不到释放

## 逃逸分析

> [技术干货 | 理解 Go 内存分配 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1861429)

逃逸分析就是指程序在编译阶段根据代码中的数据流，对代码中**变量在栈上还是堆上进行分配进行静态分析**的方法

Go 语言编译器**当发现变量的作用域没有跑出函数范围，就可以在栈上，反之则必须分配在堆**
如果函数外部没有引用，则优先放到栈中
如果函数外部存在引用，则必定放到堆中

Go 语言的逃逸分析遵循以下两个不变性：
1. **指向栈对象的指针不能存在于堆中**
2. **指向栈对象的指针不能在栈对象回收后存活**

函数默认在栈上运行、声明临时变量并分配内存，堆上动态分配内存比栈上静态分配内存开销大，应尽量分配内存至栈(减少逃逸)以减轻垃圾回收的压力

**常见的逃逸情况**
- 函数返回一个变量的指针，则该变量发生逃逸
- ~~函数参数为 interface 类型时，编译期间无法确定参数变量类型及大小，该参数变量发生逃逸~~ 调用 reflect.Valueof()处理变量时会导致变量逃逸至堆上
- 闭包逃逸：闭包函数也是一个指针，所以闭包内引用的局部变量会发生逃逸
- 将包含指针的变量(指针的指针或包含指针的结构体)传递至 channel 中，由于在编译阶段无法确定其作用域与传递的路径，一般都会逃逸
- slices 中的值是指针的指针或包含指针字段(e.g. `[]*string` )导致 slice 的逃逸。即使切片的底层存储数组仍可能位于堆栈上，数据的引用也会转移到堆中
- slice 扩容时如果其底层存储必须基于仅在运行时确定的数据进行扩展，则它将逃逸至堆上
- 调用接口类型的方法：接口类型的方法调用是动态调度，实际使用的具体实现只能在运行时确定(e.g.考虑一个接口类型为 `io.Reader` 的变量 r，对r.Read(b)的调用将导致 r 的值和字节片 b 的后续转义并因此分配到堆上)
- 栈空间不足引发逃逸：数据超过栈的大小 (Linux:64kb)

**查看程序逃逸情况**
`go build -gcflag '-m' main.go` 或 `go tool compile "-m" main.go`

## 内存对齐
