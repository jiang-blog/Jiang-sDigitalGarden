---
{"dg-publish":true,"permalink":"/Code/6.Go/Go.1d.Go内存管理/","title":"Go内存管理","noteIcon":""}
---


#  Go内存管理

##  TCMalloc

TCMalloc(Thread Cache Malloc)为每个Thread预分配一块缓存ThreadCache，每个Thread在申请内存时首先会先从ThreadCache申请，此外所有ThreadCache缓存共享一个叫CentralCache的中心缓存

当ThreadCache的缓存不足时，就会从CentralCache获取，当ThreadCache的缓存充足或者过多时，则会将内存退还给CentralCache
CentralCache由于共享，访问需要加锁；ThreadCache作为线程独立的第一交互内存，访问无需加锁

当CentralCache没有足够内存时会从一个全局共享内存堆PageHeap取，当CentralCache内存过多或者充足，则将低命中内存块退还PageHeap
Thread需要申请的大对象超过Cache容纳的内存块单元大小，也会直接从PageHeap获取

> ![image.png](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133032054-ea888b96-0fb0-46ea-ac26-4c38abc2b66f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_89%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10%2Fresize%2Cw_1038%2Climit_0)

TCMalloc将虚拟内存空间划分为多份同等大小的**Page**，每个Page默认是8KB

多个连续的Page称之为一个**Span**，每个Span记录了第一个起始Page的编号Start，和一共有多少个连续Page的数量Length，Span集合是以双向链表的形式构建，**TCMalloc以Span为单位向操作系统申请内存**

在256KB以内的小对象，TCMalloc会将这些小对象集合根据Page Size划分成多个内存刻度，同属于一个刻度类别下的内存集合称之为属于一个**Size Class**
| **对象** | **容量**     |
| -------- | ------------ |
| 小对象   | (0,256KB/32Page]    |
| 中对象   | (256KB, 1MB/128Page] |
| 大对象   | (1MB, +∞)    |

>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133255704-7c07cb59-d879-468f-a925-d3494454cb7d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_82%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)
>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133299709-8c33bad3-a31f-4844-b07c-ad54a0dc64d4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_67%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###  对象分配

TCMalloc的小对象分配
>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133672724-0ac13b26-1623-444a-8c81-0b2120b2e2fa.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_80%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

中对象分配
>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133803693-486e9b4a-ffb1-4932-a989-1df013b601c1.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_71%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

大对象分配
>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651133987470-28f3feb2-8a9e-45be-a41b-596b1bd54e8d.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_81%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

##  Golang堆内存管理

> [9、一站式Golang内存管理洗髓经 (yuque.com)](https://www.yuque.com/aceld/golang/qzyivn#VyQ8I)
> [Go 语言内存分配器的实现原理 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/)

###  Page&Object

Golang内存管理借鉴了TCMalloc的设计思想
>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134285363-999c7495-7834-4785-a6ea-c44b4615ff19.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_58%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

各层级间内存交换的单位如下：
>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134740690-c1fc15a5-af2a-474c-adaa-ddc4bb05a5e3.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_89%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

**Page**的大小依然是8KB，Page是Golang内存管理与操作系统交互衡量内存容量的基本单元
**Object**是Golang内存管理内部本身用来给对象存储内存的基本单元，一个Span在初始化时会被分成多个Object
**Object Size**指协程应用逻辑一次向Golang内存申请的Object对象大小

###  MSpan

Span的名称改为**mspan**，依然表示一组连续的Page
**Size Class**在Golang内存管理中针对Object Size而非Page Size来划分内存，Golang给内存池中固定划分了66种Size Class
**Span Class**是Golang内存管理额外定义的规格属性，针对Span大小的级别来进行划分。**一个Size Class对应两个Span Class**，其中一个Span存放需要GC扫描的对象(包含指针的对象)，另一个Span存放不需要GC扫描(不包含指针)的对象

>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134377320-3f71752d-65fa-4081-a255-09c387f23a65.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_70%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

###  MCache

**MCache**对应TCMalloc的ThreadCache，但MCache绑定Golang协程调度模型GPM中的Processor，协程逻辑层从MCache上获取内存不需要加锁
MCache中对应每个Span Class存在一个mspan，每个mcache中共134个mspan，每个mspan又根据相应的Object Size分为多个object

>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651134640677-2a153c96-b7e8-46bc-86f3-dfaf50087329.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_84%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

对于Span Class为0和1(对应Size Class为0，Object Size 为0)的规格刻度内存，MCache实际上没有分配任何内存。Golang内存管理对内存为0的数据申请特殊处理，直接返回一个固定内存地址**zerobase**

>在Golang中如\[0\]int、 struct{}所需要大小均是0，这也是为什么很多开发者在通过Channel做同步时，发送一个struct{}数据，因为不会申请任何内存，能够适当节省一部分内存空间

####  Tiny空间

每个mcache中包含一个特殊的**Tiny空间**，其从Object Size为16B的mspan中获取一个object作为tiny对象的分配空间，只有当该object中的所有对象都需要被回收时，整片内存才可能被回收

###  MCentral

MCache中出现Span Class中Span空缺时，会向MCentral申请对应的mspan，向MCentral申请mspan需要加锁

MCentral这层数据管理中实际上有134个mcentral小内存管理单元，每个mcentral对应一个Span Class，每个mcentral包含两个mspan集合，分别维护空闲mspan和非空闲mspan

###  MHeap

MCentral中的Span不够时会向MHeap申请，同样需要加锁

MHeap以Page为内存单元进行管理，用来详细管理每一系列Page的结构称之为一个**HeapArena**，一个HeapArena占用内存64MB，里面的内存以page方式存放，所有的HeapArena组成的集合是一个**Arenas**

###  对象分配流程

Golang将对象分为tiny对象，小对象及大对象

**大量的微小对象可能会使Object Size = 8B的mspan产生许多空间浪费**。所以Golang内存管理决定将小于16B的内存申请统一归类为Tiny对象申请
> ![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135299397-38b04b81-3179-44e4-81ca-bb476b69f22f.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_73%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

对于**16B至32KB**的内存申请，Golang采用小对象的分配流程

>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135402859-c6404c6a-f0fd-4bb9-bbef-0c5286b0b2a4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_100%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

对于大于32KB的大对象申请直接从MHeap中分配，Proceesor直接向MHeap申请对象所需要的适当Pages

>![](https://cdn.nlark.com/yuque/0/2022/png/26269664/1651135463380-9ee93382-7deb-48c0-ab38-519d679101e4.png?x-oss-process=image%2Fwatermark%2Ctype_d3F5LW1pY3JvaGVp%2Csize_90%2Ctext_5YiY5Li55YawQWNlbGQ%3D%2Ccolor_FFFFFF%2Cshadow_50%2Ct_80%2Cg_se%2Cx_10%2Cy_10)

##  Golang栈内存管理

> [Go 语言的栈内存和逃逸分析 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/)

Golang在栈内存管理中采用**连续栈**，其核心原理是每当程序的栈空间不足时，初始化一片更大的栈空间并将原栈中的所有值都迁移到新栈中，新的局部变量或者函数调用就有充足的内存空间。

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

##  虚拟内存布局

>[Go 语言内存分配器的实现原理 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-memory-allocator/#711-%E8%AE%BE%E8%AE%A1%E5%8E%9F%E7%90%86)

###  线性内存

V1.10在启动时会初始化整片虚拟内存区域，为 `spans`、`bitmap` 和 `arena` 三个区域分别预留了 512MB、16GB 以及 512GB 的内存空间
- `spans` 区域存储了指向内存管理单元mspan的指针，每个内存单元会管理几页的内存空间，每页大小为 8KB；
- `bitmap` 用于标识 `arena` 区域中保存了对象的地址，bitmap中的每个字节都会表示堆区中的 32 字节是否空闲；
- `arena` 区域是真正的堆区，运行时会将 8KB 看做一页，这些内存页中存储了所有在堆上初始化的对象；

###  稀疏内存

V1.11运行时使用二维的 `heapArena`数组管理所有的内存，每个单元都会管理 64MB 的内存空间

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

该结构体中的 `bitmap` 和 `spans` 与线性内存中的 `bitmap` 和 `spans` 区域一一对应，`zeroedBase` 字段指向了该结构体管理的内存的基地址。上述设计将原有的连续大内存切分成稀疏的小内存，而用于管理这些内存的元信息也被切成了小块

> 不同平台和架构的二维数组大小可能完全不同，如果我们的 Go 语言服务在 Linux 的 x86-64 架构上运行，二维数组的一维大小会是 1，而二维大小是 4,194,304，因为每一个指针占用 8 字节的内存空间，所以元信息的总大小为 32MB。由于每个`heapArena`会管理 64MB 的内存，整个堆区最多可以管理 256TB 的内存，这比之前的 512GB 多好几个数量级

##  逃逸分析

维基百科：
> 在编译程序优化理论中，**逃逸分析**是一种确定[指针](https://zh.wikipedia.org/wiki/%E6%8C%87%E6%A8%99_(%E9%9B%BB%E8%85%A6%E7%A7%91%E5%AD%B8) "指标 (计算机科学)")动态范围的方法——分析在程序的哪些地方可以访问到指针。

换句话说，逃逸分析就是指程序在编译阶段根据代码中的数据流，对代码中**变量在栈上还是堆上进行分配进行静态分析**的方法

Go 语言的逃逸分析遵循以下两个不变性：
1. 指向栈对象的指针不能存在于堆中
2. 指向栈对象的指针不能在栈对象回收后存活

函数默认在栈上运行、声明临时变量并分配内存，堆上动态分配内存比栈上静态分配内存开销大，应尽量分配内存至栈(减少逃逸)以减轻垃圾回收的压力

**常见的逃逸情况**
- 函数返回一个变量的指针，则该变量发生逃逸
- 函参为interface类型时编译期间无法确定参数变量类型及大小，该参数变量发生逃逸
- 闭包逃逸：闭包函数也是一个指针，所以闭包内引用的局部变量会发生逃逸
- 将包含指针的变量(指针的指针或包含指针的结构体)传递至channel中，由于在编译阶段无法确定其作用域与传递的路径，一般都会逃逸
- 栈空间不足引发逃逸：数据超过栈的大小 (Linux:64kb)

查看程序逃逸情况 `go build -gcflag '-m' main.go`
