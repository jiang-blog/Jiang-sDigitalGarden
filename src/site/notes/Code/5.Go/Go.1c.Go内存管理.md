---
{"dg-publish":true,"permalink":"/Code/5.Go/Go.1c.Go内存管理/","title":"Go内存管理","noteIcon":""}
---


# Go 内存管理

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

> ![|675](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134285363-999c7495-7834-4785-a6ea-c44b4615ff19.png)
>![|675](https://img.draveness.me/2020-02-29-15829868066479-go-memory-layout.png)

### 内存单位

各层级间内存交换的单位如下：

>![|600](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134740690-c1fc15a5-af2a-474c-adaa-ddc4bb05a5e3.png)

#### Object

*Object* 是 Go 内存管理内部用来存储对象内存的基本单元

*Object Size* 指协程一次向 Golang 内存申请的 Object 大小

在 Go 内存管理中，针对 Object Size 固定划分了67种 **Size Class**，每一个 Size Class 都会存储特定大小的 Object 并且包含特定数量的 Page 以及 Object，最大的 Size Class 为 32KB

| class | bytes/object | bytes/span | objects | tail waste | max waste |
|:-----:|:------------:|:----------:|:-------:|:----------:|:---------:|
|   1   |      8       |    8192    |  1024   |     0      |  87.50%   |
|   2   |      16      |    8192    |   512   |     0      |  43.75%   |
|   3   |      24      |    8192    |   341   |     0      |  29.24%   |
|   4   |      32      |    8192    |   256   |     0      |  46.88%   |
|   5   |      48      |    8192    |   170   |     32     |  31.52%   |
|   6   |      64      |    8192    |   128   |     0      |  23.44%   |
|   7   |      80      |    8192    |   102   |     32     |  19.07%   |
|   …   |      …       |     …      |    …    |     …      |     …     |
|  67   |    32768     |   32768    |    1    |     0      |  12.50%   |

Go 内存管理对内存为0的数据申请特殊处理，直接返回一个固定内存地址 **zerobase**

> 在 Go 中如\[0\]int、 struct{}所需要的空间大小大小均是0，这也是为什么很多开发者在通过 Channel 做同步时，发送一个 struct{}数据，因为不会申请任何内存，能够适当节省一部分内存空间

此外 runtime 中还包含 object size 为 0 的特殊 Size Class，它能够管理大于 32KB 的特殊对象

#### Page

*Page*的大小是8KB，是 Golang 内存管理系统与操作系统交互衡量内存容量的基本单元

#### MSpan

与 TCMalloc 一样，mspan 包含一个及以上连续的 Page

**对应每个 Size Class 存在两个 Span Class**，一个存放需要 GC 扫描的对象(包含指针的对象)，另一个存放不需要 GC 扫描(不包含指针)的对象
因此 Span class 共有 68 * 2 = 136 个，每种 class 的 mspan 根据相应的 Object Size 分为 n(>=1)个 Object

mspan 相互链接构成一个双向链表
```go
type mspan struct {
  // 双向链表
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
  state     mSpanStateBox // 存储mspan的状态
	...
}
```
`spanClass` 是一个 uint8 类型的整数，它的前 7 位存储着 Span class 的 ID，最后一位表示是否包含指针
runtime 会使用 `runtime.mSpanList` 存储双向链表的头结点和尾节点并在 MCache 以及 MCentral 中使用

mspan 的状态包括四种
- `mSpanDead`
- `mSpanInUse` - 已被分配
- `mSpanManual` - 已被分配，手动管理
- `mSpanFree` - 处于空闲堆

当用户程序或者线程向 mcache 中对应的 mspan 申请内存时，mspan 使用 `allocCache` 字段以 object 为单位在管理的内存中快速查找待分配的空间
如果能在内存中找到空闲的内存单元则直接返回，当内存中不包含空闲的内存时，MCache 会为调用 `runtime.mcache.refill` 更新内存管理单元以满足为更多对象分配内存的需求

#### MCache

MCache 对应 TCMalloc 的 ThreadCache，是 Go 中的线程缓存
Go 协程调度模型 GPM 中的**每个调度器 P 都有自己独立的一个 mcache**，因此**协程从 MCache 获取内存不需要加锁**

每一个 mcache 中对应每个 Span Class 都有相应的一个 mspan，因此每一个 mcache 中的 mspan 有 136 个，每个 mspan 又根据相应的 Object Size 分为 n(>=1) 个 object，这些 mspan 都存储在 mcache 结构体的 `alloc` 字段中
```go
type mcache struct {
  ...
	alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass
	...
}
```

**MCache 在刚刚被初始化时不包含 mspan 实例**，所有 mspan 都为空占位符 `emptymspan`，当用户程序申请内存时 MCache 从上一级组件 MCentral 中获取新的 mspan 以满足内存分配需求
mcache 通过 `runtime.mcache.refill` 获取一个指定 Span Class 的 mspan，被替换的 mspan 不能包含空闲内存空间，而获取的 mspan 中需要至少包含一个空闲 object 用于分配内存

>![|900](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134640677-2a153c96-b7e8-46bc-86f3-dfaf50087329.png)
> **该图 Span Class 部分有误，134 个 span class 中不包括 object size 为 0 的 span class**

##### Tiny 空间

每个 mcache 中包含一个特殊的 **Tiny 空间**，其从 Object Size 为16B 的 mspan 中获取一个 object 作为 tiny 对象的分配空间，只有当该 object 中的所有对象都需要被回收时，整片内存才可能被回收

**大量的微小对象可能会使 Object Size = 8B 的 mspan 产生许多空间浪费**，所以 Go将小于16B 的内存申请统一归类为 Tiny 对象申请

下面的这三个字段组成了微对象分配器，专门管理 16 B以下的对象
```go
type mcache struct {
	tiny       uintptr // 一个堆指针，因为mcache在非GC内存中，所以在标记终止时在RelaseAll中进行清除
	tinyoffset uintptr // 下一个空闲内存所在的偏移量
	tinyAllocs uintptr // 分配的tiny对象数量
	...
}
```
**微分配器只会用于分配非指针类型的内存**，上述三个字段中 `tiny` 会指向堆中的一片内存，`tinyOffset` 是下一个空闲内存所在的偏移量，最后的 `local_tinyallocs` 会记录内存分配器中分配的对象个数

#### MCentral

MCache 中某个 mspan 空缺或用尽时，会向 MCentral 申请对应的 mspan，**向 MCentral 申请 mspan 需要加互斥锁**

根据 Span Class，MCentral 具体分为 136 个 mcentral，每个 mcentral 都会管理对应 Span Class 的 mspan
mcentral 会同时持有两个 mspan 的集合，分别存储包含空闲对象和不包含空闲对象的 mspan
每个集合又分别存放已经清扫完的 mspan 和未清扫完的 mspan，这两个角色在每次 GC 循环中交换

```go
type mcentral struct {
	spanclass spanClass
	partial  [2]spanSet // 包含空闲object的mspan
	full     [2]spanSet // 没有空闲object的mspan
}
```

MCentral 通过  `runtime.mcentral.cacheSpan` 方法向 mcache 提供新的 mspan，分为以下几步：
1. 调用 `runtime.mcentral.partialSwept` 从清理过的、包含空闲 obejct 的 `spanSet` 结构中查找可以使用的 mspan
2. 调用 `runtime.mcentral.partialUnswept` 从未被清理过的、有空闲 obejct 的 `spanSet` 结构中查找可以使用的 mspan
3. 调用 `runtime.mcentral.fullUnswept` 从未被清理的、不包含空闲 obejct 的 `spanSet` 中获取 mspan，并通过 `runtime.mspan.sweep` 清理它的内存空间
4. 调用 `runtime.mcentral.grow` 从 mheap 中申请新的 mspan
    1. 根据预先计算的 `class_to_allocnpages` 和 `class_to_size` 获取待分配的 page 数以及 span class 并调用 `runtime.mheap.alloc` 获取新的 `runtime.mspan` 结构
5. 更新 mspan 的 `allocBits` 和 `allocCache` 等字段，帮助快速分配内存

MCentral 中的 mspan 不够时，会采用 `runtime.mcentral.grow` 扩容方法根据预先计算的 `class_to_allocnpages` 和 `class_to_size` 获取待分配的 Pages 以及 Size Class ，并调用 `runtime.mheap.alloc` 获取新的 mspan

#### MHeap

MCentral 中的 mspan 不够时会向 MHeap 申请，**同样需要加锁**

MHeap 是内存分配的核心结构体，是堆内存的抽象，把从系统申请出的内存页组织成 mspan，并保存起来

MHeap结构体中包含两组非常重要的字段，其中一个是全局的中心缓存列表 `central`，另一个是管理堆区内存区域的 `arenas` 以及相关字段
- MHeap 中包含一个长度为 136 的 `runtime.mcentral` 数组，其中 68 个为需要 GC 扫描的 mcentral，另外的 68 个是不需要 GC 扫描的 mcentral
- MHeap 以 Page 为内存单元进行管理，用来详细管理每一组 Page 的结构称之为一个 **heapArena**，一个 heapArena 占用内存64MB，里面的内存以 page 方式存放，**Arenas** 是所有的 heapArena 组成的集合

> ![|600](https://img.draveness.me/2020-02-29-15829868066531-mheap-and-memories.png)

### 对象分配流程

堆上所有的对象都会通过调用 `runtime.newobject` 函数分配内存，该函数会调用 `runtime.mallocgc` 分配指定大小的内存空间

`runtime.mallocgc` 将对象分为微对象，小对象及大对象，根据对象的大小执行不同的分配逻辑

对于小于16B 的非指针对象内存申请，Go 采用 Tiny 对象的分配流程

> ![|750](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135299397-38b04b81-3179-44e4-81ca-bb476b69f22f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_73%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

对于**16B 至32KB**的内存申请，Go 采用小对象的分配流程
1. 确定分配对象的大小以及跨度类 `runtime.spanClass`
2. 从 mcache、mcentral 或者 mheap 中获取 mspan 并从中找到空闲的内存空间
3. 调用 `runtime.memclrNoHeapPointers` 清空空闲内存中的所有数据

>![|750](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135402859-c6404c6a-f0fd-4bb9-bbef-0c5286b0b2a4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_100%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

对于大于32KB 的大对象，Proceesor 直接调用 `runtime.mcache.allocLarge` 向 MHeap 申请对象所需要的适当 Pages
申请内存时会创建一个 Span Class 为 0 的 `runtime.spanClass` 并调用 `runtime.mheap.alloc` 分配一个管理对应内存的 mspan

>![|750](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135463380-9ee93382-7deb-48c0-ab38-519d679101e4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_90%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

### 虚拟内存布局

>[Go 语言内存分配器的实现原理 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/#711-%E8%AE%BE%E8%AE%A1%E5%8E%9F%E7%90%86)

#### 线性内存

> ![heap-before-go-1-10.png (1207×330) (draveness.me)](https://img.draveness.me/2020-10-19-16031147347484/heap-before-go-1-10.png)

V1.10 以前 Go 在启动时会初始化整片虚拟内存区域，为 `spans`、`bitmap` 和 `arena` 三个区域分别预留了 512MB、16GB 以及 512GB 的内存空间
- `spans` 区域存储了指向内存管理单元 mspan 的指针，每个内存单元会以页为单位管理内存空间，每页大小为 8KB
- `bitmap` 用于标识 `arena` 区域中保存了对象的地址，bitmap 中的每个字节都会表示堆区中的 32 字节是否空闲
- `arena` 区域是真正的堆区，运行时会将 8KB 看做一页，这些内存页中存储了所有在堆上初始化的对象

对于任意一个地址，都可以根据 arena 的基地址计算该地址所在的页数并通过 `spans` 数组获得管理该片内存的管理单元 `mspan`，spans 数组中多个连续的位置可能对应同一个 `mspan` 结构
Go 语言在垃圾回收时会根据指针的地址判断对象是否在堆中并找到管理该对象的 `mspan`

线性内存在 C 和 Go 混合使用时会导致程序崩溃：
- 分配的内存地址会发生冲突，导致堆的初始化和扩容失败
- 没有被预留的大块内存可能会被分配给 C 语言的二进制，导致扩容后的堆不连续

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

所有的内存最终都从操作系统中申请，所以 Go 语言的运行时构建了操作系统的内存管理抽象层，该抽象层将运行时管理的地址空间分成以下四种状态：
|    状态    | 解释                                                                                          |
|:----------:| --------------------------------------------------------------------------------------------- |
|   `None`   | 内存没有被保留或者映射，是地址空间的默认状态                                                  |
| `Reserved` | 运行时持有该地址空间，但是访问该内存会导致错误                                                |
| `Prepared` | 内存被保留，一般没有对应的物理内存，访问该片内存的行为是未定义的，可以快速转换到 `Ready` 状态 |
|  `Ready`   | 可以被安全访问                                                                                |

每个不同的操作系统都会包含一组用于管理内存的特定方法，这些方法可以让内存地址空间在不同的状态之间转换

## Go 栈内存管理

> [Go 语言的栈内存和逃逸分析 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/)
> [解密Go协程的栈内存管理-腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1700947)

每个 goroutine 都维护着一个只能自己使用的栈区，栈区的初始大小是 **2KB**，在 goroutine 运行的时候栈区会按照需要增长和收缩，占用的内存最大限制的默认值在 64 位系统上是 **1GB**

### 栈扩容缩容

go V1.3 以前使用**分段栈**，栈空间不足时开辟一个新的栈空间，多个不连续的栈空间以双向链表的形式串联起来
分段栈存在**热分裂问题**：当一个 goroutine 的栈空间接近用尽时，任意的函数调用都会触发栈的扩容，当函数返回后又会触发栈的收缩，如果在一个循环中调用函数，栈的分配和释放就会造成巨大的额外开销

之后 Go 在栈内存管理中采用**连续栈**，每当程序的栈空间不足时，初始化一片原栈两倍大小的新栈空间，并将原栈中的所有内容都迁移到新栈中
1. 调用用 `runtime.newstack` 在内存空间中分配更大的栈内存空间
2. 使用`runtime.copystack`将旧栈中的所有内容复制到新的栈中
3. **将指向旧栈对应变量的指针重新指向新栈**(指向栈对象的指针都在栈上)
4. 调用`runtime.stackfree`销毁并回收旧栈的内存空间

因为需要拷贝变量和调整指针，连续栈增加了栈扩容时的额外开销，但是通过合理栈缩容机制就能避免热分裂带来的性能问题
缩容机制：在 GC 期间如果 Goroutine 使用了栈内存的四分之一，那就将其内存减少一半容量，新栈的大小不能低于程序的最低限制 2KB

程序会在几乎所有的函数调用之前检查当前 Goroutine 的栈内存是否充足

### 栈空间结构

Go 语言中的执行栈由 `runtime.stack` 表示，该结构体中只包含两个字段，分别表示栈的顶部和栈的底部，每个栈结构体都表示范围为 \[lo, hi) 的内存空间：
```go
type stack struct {
	lo uintptr
	hi uintptr
}
```

栈空间在运行时中包含两个重要的全局变量 `runtime.stackpool` 和 `runtime.stackLarge`，分别表示全局的栈缓存和大栈缓存，前者用于分配小于 32KB 的栈空间，后者用来分配大于 32KB 的栈空间
```go
var stackpool [_NumStackOrders]struct {
	item stackpoolItem
	_    [cpu.CacheLinePadSize - unsafe.Sizeof(stackpoolItem{})%cpu.CacheLinePadSize]byte
}

type stackpoolItem struct {
	mu   mutex
	span mSpanList
}

var stackLarge struct {
	lock mutex
	free [heapAddrBits - pageShift]mSpanList
}
```

两个全局变量内部都与 `mspan` 有关，可以认为 Go 语言的栈内存都是分配在堆上的
但如果 runtime 只使用全局变量分配内存，会造成线程之间的锁竞争进而影响程序的执行效率，由于栈内存与线程关系比较密切，所以在每一个线程缓存 `mcache` 中都加入了栈缓存 `stackcache` 减少锁竞争影响
```go
type mcache struct {
	stackcache [_NumStackOrders]stackfreelist
}

type stackfreelist struct {
	list gclinkptr
	size uintptr
}
```

### 栈空间分配

根据线程缓存和申请栈的大小，栈空间通过以下方式分配：
- 申请栈空间小于32KB
  1. 先从 `mcache` 中的栈缓存 `stackcache` 中分配
  2. 如果 `stackcache` 内存不足，则从全局栈缓存 `stackpool` 中分配
  3. 如果 `stackpool` 内存不足，则从堆 `mheap` 中分配
- 申请栈空间大于32KB
  1. 直接从全局栈内存缓存池 `stackLarge` 中分配
  2. 全局栈内存缓存池 `stackLarge` 不足，则从堆 `mheap` 分配

## 内存泄漏

内存泄漏就是程序生命周期中一些对象不能被及时回收，一直占用着内存，导致这部分内存不可用的情况

Go语言中仍可能发生内存泄漏：预期的能很快被释放的内存由于附着在了长期存活的内存上、或生命期意外地被延长，导致预计能够立即回收的内存长时间得不到回收。

1. 预期能被快速释放的内存因被根对象引用而没有得到迅速的释放
e.g. 当有一个全局对象时，可能不经意间将某个变量附着在其上，且忽略释放该变量，则其内存永远不会得到释放
2. Goroutine泄漏
Goroutine 在运行过程中消耗的用于维护上下文信息的内存不会被释放，当一个程序持续不断地产生新的 Goroutine、且不结束已创建的 Goroutine 并复用内存时就会造成内存泄露
3. Channel 泄漏
如果一个 goroutine 阻塞在 channel 处而 channel 数据一直未改变，则该 goroutine 会被动永久休眠，整个 goroutine 及其执行栈都得不到释放

## 逃逸分析

> [技术干货 | 理解 Go 内存分配 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1861429)

逃逸分析就是指程序在编译阶段根据代码中的数据流，对代码中**变量在栈上还是堆上进行分配进行静态分析**的方法

Go 语言编译器**当发现变量的作用域没有离开函数范围，就可以在栈上，反之则必须分配在堆**
如果函数外部没有引用，则优先放到栈中
如果函数外部存在引用，则必定放到堆中

Go 语言的逃逸分析遵循以下两个不变性：
1. **指向栈对象的指针不应存在于堆中**
2. **指向栈对象的指针不应在栈对象回收后存活**

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
