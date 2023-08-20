---
{"dg-publish":true,"permalink":"/Code/5.Go/Go.0b.Go标准库/","title":"Go 标准库","noteIcon":""}
---


# Go 标准库

## Runtime

### runtime.SetFinalizer

go提供了`runtime.SetFinalizer`函数，当GC准备释放对象时，会回调该函数指定的方法

## Sync

> [Go 语言并发编程、同步原语与锁 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/#rwmutex)

### Sync.Map

>  [深度解密 Go 语言之 sync.map - Stefno - 博客园 (cnblogs.com)](https://www.cnblogs.com/qcrao-2018/p/12833787.html)

原生 map 和 slice 非线程安全

Sync.Map 为并发安全的哈希表
```Go
// https://github.com/golang/go/blob/master/src/sync/map.go

type Map struct {
 mu Mutex
 read atomic.Pointer[readOnly]
 dirty map[interface{}]*entry
 misses int
}

// read表结构
type readOnly struct {
	m       map[interface{}]*entry
	amended bool // dirty中包含read没有的数据时为true
}

// read和dirty中元素都为指针
type entry struct {
	p atomic.Pointer[interface{}]
}
```
元素 p 有三种状态
- 当 `p == nil` 时，说明这个键值对已被删除，并且 `m.dirty == nil` 或 `m.dirty[k]` 指向该 entry
- 当 `p == expunged` 时，说明这条键值对已被删除，并且`m.dirty != nil`且 m.dirty 中没有这个 key
- 其他情况，p 指向一个正常的值，表示实际 `interface{}` 的地址

Sync.Map 底层**使用了两个原生 map 进行读写分离**，读数据存在只读哈希表 read 上，最新写入的数据则存在 dirty 哈希表上
read 和 dirty map 中相同 key 的 entry 是同一个

优点：通过读写分离降低锁时间来提高效率
缺点：不适用于大量写的场景

#### 插入

```go
func (m *Map) Store(key, value interface{}) {
    read, _ := m.read.Load().(readOnly)
    // 如果 read 里存在，则尝试存到 entry 里
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    // 如果上一步没执行成功，则要分情况处理
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    // 和 Load 一样，重新从 read 获取一次
    if e, ok := read.m[key]; ok {
        // 情况 1：read 里存在
        if e.unexpungeLocked() {
            // 如果 p == expunged，则需要先将元素插入到dirty
            m.dirty[key] = e
        }
        // 用值更新entry
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        // 情况 2：read 里不存在，但 dirty 里存在，则用值更新 entry
        e.storeLocked(&value)
    } else {
        // 情况 3：read 和 dirty 里都不存在
        if !read.amended { // dirty中不含read没有的元素
            // 如果dirty为空则刷新dirty
            m.dirtyLocked()
            // 然后将amended改为 true
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        // 将新的键值存入 dirty
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}

func (e *entry) tryStore(i *interface{}) bool {
    for {
        p := atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
    }
}

func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

func (e *entry) storeLocked(i *interface{}) {
    atomic.StorePointer(&e.p, unsafe.Pointer(i))
}

func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }

    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    for k, e := range read.m {
        // 判断 entry 是否被删除，否则就存到 dirty 中
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    for p == nil {
        // 如果有 p == nil(即键值对被 delete)，则会在这个时机被置为 expunged
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
```
1. 尝试直接通过原子操作更新 read 中的 key
2. 如更新失败，加锁继续插入
    1. Read 中存在该 key
        1. 如果对应 entry 被标记为 `expunged` ，表示 dirty 中不包含该 key 且 `dirty != nil`，先添加该 key 至 dirty
        2. 更新 read 中的 key
    2. Read 中不存在但 Dirty 中存在该 key，仅更新 dirty 中的 key
    3. 都不存在
       1. 若 dirty 为空，触发一次 dirty 刷新，新建一个 dirty， **遍历复制** read 中**未被删除的元素**到 dirty 中
       2. 更新 `amended` 为 true，标识 dirty map 中存在 read map 中没有的 key
       3. 将新元素仅添加至 dirty

#### 读取

```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    // 首先尝试从 read 中读取 readOnly 对象
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    
    // 如果不存在且dirty有read没有的数据,则尝试从dirty中获取
    if !ok && read.amended {
        m.mu.Lock()
        // 由于上面 read 获取没有加锁，为了安全再检查一次
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]

        // 确实不存在且dirty中有新数据,则从 dirty 获取
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // 标记一次穿透
            m.missLocked()
        }
        m.mu.Unlock()
    }

    // amended为false代表dirty为nil
    if !ok {
        return nil, false
    }
    // 从 entry.p 读取值
    return e.load()
}

func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    // 当 miss 积累过多，会将 dirty 存入 read，然后 将 amended = false，且 m.dirty = nil
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
```
1. 首先通过原子操作查询 read
2. 不存在时加锁再次查询 read
3. 依然不存在时如果 dirty 和 read 数据不一致，查询 dirty

通过 `misses` 字段统计穿透 read 读取 dirty 的次数，超过阈值(`len(dirty)`)代表穿透概率过大，则**使用 dirty 数据覆盖 read**，之后将 dirty 设为 nil

#### 删除

```go
func (m *Map) Delete(key interface{}) {
    m.LoadAndDelete(key)
}

// LoadAndDelete 作用等同于 Delete，并且会返回值与是否存在
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
    // 获取逻辑和 Load 类似
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    // read 不存在且dirty数据不一致则查询 dirty
    if !ok && read.amended {
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // 直接删除dirty中key
            delete(m.dirty, key)
            m.missLocked()
        }
        m.mu.Unlock()
    }
    // 查询到 entry 后执行伪删除
    if ok {
        // 将entry.p标记为nil，数据并没有实际删除
        // 真正删除数据并被置为 expunged，是在 Store 的 tryExpungeLocked 中
        return e.delete()
    }
    return nil, false
}

func (e *entry) delete() (hadValue bool) {
    for {
        // 再次加载数据的指针，如果指针为空或已被标记删除，那么返回false，删除失败
        p := atomic.LoadPointer(&e.p)
        if p == nil || p == expunged {
            return false
        }
        
        // 原子操作
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return true
        }
    }
}
```
删除时的查找流程与读取相同
1. read 中存在该 key，对 read 采用*延迟删除*，仅将 read 中目标 key 对应的 entry 值置为 nil
2. 若仅 dirty 中存在该 key，直接删除

Read 中已经被删除 key 对应的 entry 首先被标记为 nil
当下一次 dirty 刷新时再次更新为 `expunged`，不被刷新至 dirty 中
下一次 dirty 覆盖 read 时被回收

标记的意义在于若两个 map 都存在这个 key，仅做标记删除可以在下次查找这个 key 时，命中 read，提升效率。若只有在 dirty 中存在时，read 起不到缓存的作用，直接删除

### Sync.Mutex

> [Go精妙的互斥锁设计 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343563412)
> [GO 互斥锁实现原理剖析_weixin_34208283的博客-CSDN博客](https://blog.csdn.net/weixin_34208283/article/details/91699624?app_version=5.15.4&code=app_1562916241&csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%2291699624%22%2C%22source%22%3A%22weixin_44764530%22%7D&uLinkId=usr1mkqgl919blen&utm_source=app)

互斥锁
加锁：`sync.Mutex.Lock`
解锁：`sync.Mutex.Unlock`

**加锁**
1. 如果锁是完全空闲状态，则通过 CAS 直接加锁
2. 如果锁处于正常模式，则会尝试自旋，**持有 CPU** 等待锁的释放
3. 如果当前 goroutine 不再满足自旋条件，则会计算锁的期望状态，并尝试更新锁状态
4. 在更新锁状态成功后，会判断当前 goroutine 是否能获取到锁，能获取锁则直接退出
5. 当前 goroutine 不能获取到锁时，则会由 sleep 原语 `SemacquireMutex` 陷入睡眠，等待解锁的 goroutine 发出信号进行唤醒
6. 唤醒之后的 goroutine 发现锁处于饥饿模式，则能直接拿到锁，否则重置自旋迭代次数并标记唤醒位，重新进入步骤2中

**解锁**

- 如果通过原子操作 `AddInt32` 后，锁变为完全空闲状态，则直接解锁
- 如果解锁一个没有上锁的锁，则直接抛出异常
- 如果锁处于正常模式，且没有 goroutine 等待锁释放，或者锁被其他 goroutine 设置为了锁定状态、唤醒状态、饥饿模式中的任一种(非空闲状态)，则会直接退出；否则，会通过 wakeup 原语 `Semrelease` 唤醒 waiter
- 如果锁处于饥饿模式，会直接将锁的所有权交给等待队列队头 waiter，唤醒的 waiter 会负责设置 `Locked` 标志位

```go
type Mutex struct {
    state int32    // 表示当前互斥锁的状态
    sema  uint32   // 用于控制锁状态的信号量
}

const (
    mutexLocked = 1 << iota
    mutexWoken
    mutexStarving
    mutexWaiterShift = iota   // mutexWaiterShift值为3，通过右移3位的位运算，可计算waiter个数
    starvationThresholdNs = 1e6 // 1ms，进入饥饿状态的等待时间
)
```

`state` 字段表示当前互斥锁的状态信息，它是 `int32` 类型，其低三位的二进制位均有相应的状态含义
- `mutexLocked` 是 `state` 中的低1位，代表该互斥锁是否被加锁
- `mutexWoken` 是低2位，代表互斥锁上是否有被唤醒的 goroutine
- `mutexStarving` 是低3位，代表当前互斥锁是否处于饥饿模式
- `state` 剩下的前29位用于统计在互斥锁上的等待队列中 goroutine 数目(`waiter`)

#### 饥饿模式

饥饿模式是在 V1.9版本引入的优化，目的是保证互斥锁的公平性

正常模式下，锁的等待者会按照先进先出的顺序获取锁
但新请求锁的 goroutine 具有优势：它正在 CPU 上执行，而且可能有好几个，所以刚刚唤醒的 goroutine 有很大可能在锁竞争中失败，长时间获取不到锁
为了减少这种情况的出现，一旦 Goroutine 超过 1ms 没有获取到锁，就会将当前互斥锁切换为饥饿模式，防止部分 Goroutine 被饿死

**在饥饿模式中，互斥锁会直接交给等待队列最前面的 Goroutine**，新的 Goroutine 在该状态下不能获取锁，也不会进入自旋状态，只会在队列的末尾等待
如果一个 Goroutine 获得了互斥锁并且它在队列的末尾或者它等待的时间少于 1ms，那么当前的互斥锁就会被切换回正常模式

相比于饥饿模式，正常模式下的互斥锁能够提供更好的性能，因为 goroutine 可以连续多次获取锁，饥饿模式能避免由于 goroutine 陷入等待无法获取锁而造成的高尾延时

#### 自旋

Goroutine 能进入自旋的条件如下
- 当前互斥锁处于正常模式
- 当前运行的机器是多核 CPU，且 GOMAXPROCS>1
- 至少存在一个其他正在运行的处理器 P，并且它的本地运行队列(local runq)为空
- 当前 goroutine 进行自旋的次数小于4

### Sync.RWMutex

读写互斥锁 sync.RWMutex 是细粒度的互斥锁，它不限制资源的并发读，但是读写、写写操作无法并行执行

```go
type RWMutex struct {
	w           Mutex  // 复用互斥锁提供的能力
	writerSem   uint32 // 写等待读
	readerSem   uint32 // 读等待写
	readerCount int32  // 读锁,表示当前正在执行的读数量
	readerWait  int32  // 当写操作被阻塞时等待的读操作个数
}
```

RWMutex 通过获取写锁时先阻塞写锁的获取后阻塞读锁的获取来保证读操作不会被连续的写操作饿死

#### 读锁

读锁加锁调用 `sync.RWMutex.RLock` 方法
```go
func (rw *RWMutex) RLock() {
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
}
```
1. 使用原子操作令 `readerCount += 1`
2. 若 `readerCount` 为负数代表已加写锁，休眠等待锁释放
3. 否则获取写锁成功返回

读锁解锁调用 `sync.RWMutex.RUnlock` 方法
```go
func (rw *RWMutex) RUnlock() {
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		rw.rUnlockSlow(r)
	}
}
```
1. 使用原子操作令 `readerCount -= 1`
2. 若 `readerCount` 为正数代表解锁成功
3. 若 `readerCount` 为负数代表有写锁正在等待，调用 `sync.RWMutex.rUnlockSlow` 方法减少获取锁的写操作等待的读操作数 `readerWait`，在所有读操作都被释放之后触发写操作的信号量 writerSem，该信号量被触发时，调度器就会唤醒尝试获取写锁的 Goroutine

#### 写锁

写锁加锁调用 `sync.RWMutex.Lock` 方法
```go
func (rw *RWMutex) Lock() {
	rw.w.Lock()
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
}
```
1. 加锁阻塞其他写锁请求
2. 将 `readerCount` 置为负数阻塞后续读锁请求
3. 已有读锁时(`readerCount` 为正数)休眠等待，前方读锁执行完成后被唤醒

写锁解锁调用 `sync.RWMutex.UnLock` 方法
```go
func (rw *RWMutex) Unlock() {
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		throw("sync: Unlock of unlocked RWMutex")
	}
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	rw.w.Unlock()
}
```
1. 将 `readerCount` 重新置为正数，释放读锁
2. 通过 for 循环唤醒所有阻塞获取读锁的 Goroutine
3. 调用 `sync.Mutex.Unlock` 释放写锁

### Sync.Once

确保在并发程序中一个函数仅执行一次

Once 常常用来初始化单例资源，或者并发访问只需初始化一次的共享资源，或者在测试的时候初始化一次测试资源

sync.Once 只暴露了一个幂等方法 Do
`sync.Once.Do(f())` 接收一个无参数无返回值的函数 f 作为输入，该函数在且仅在第一次调用 Do 时执行

Once 与 init 方法的不同在于 init 是在其所在的 package 首次加载时执行的，而 sync.Once 可以在代码的任意位置初始化和调用，是在第一次用的它的时候才会
初始化

#### 实现

```go
type Once struct {
	done uint32 // 标识是否已执行
	m    Mutex
}

func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```
once 通过锁和一个唯一变量保证仅执行一遍

### Sync.WaitGroup

Go 中使用 sync.WaitGroup 来实现并发任务的同步以及协程任务等待

```Go
type WaitGroup struct {
	noCopy noCopy     // 保证WaitGroup不会被开发者通过再赋值的方式拷贝
	state1 [3]uint32 // 存储状态和信号量
}

func (wg * WaitGroup) Add(delta int)  // 计数器加delta
func (wg *WaitGroup) Done()           // Add(-1)
func (wg *WaitGroup) Wait()           // 会阻塞代码的运行，直至计数器减为0
```

常见的使用场景是批量发出 RPC 或者 HTTP 请求

```go
func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	v := int32(state >> 32)
	w := uint32(state)
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
	if v > 0 || w == 0 {
		return
	}
	*statep = 0
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false, 0)
	}
}
```
当调用计数器归零，即所有任务都执行完成时，Add 函数会通过 `sync.runtime_Semrelease` 唤醒处于等待状态的 Goroutine

```go
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()
	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32)
		if v == 0 {
			return
		}
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			runtime_Semacquire(semap)
			if +statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			return
		}
	}
}
```
Wait 会在计数器大于 0 并且不存在等待的 Goroutine 时，调用 `runtime.sync_runtime_Semacquire` 休眠

## Time

## Atomic

```Go
type atomic interface{
  func AddT(addr *T, delta T)(new T)
  func StoreT(addr *T, val T)
  func LoadT(addr *T) (val T)
  func SwapT(addr *T, new T) (old T)
  func CompareAndSwapT(addr *T, old, new T) (swapped bool)
}
```

Atomic 提供的方法皆为原子操作
原子操作就是指这一系列的操作在 cpu 上执行时是一个不可分割的整体，要么全部执行，要么全部不执行，不会受到其他操作的影响

## Context

Context 主要用于实现 goroutine 之间的**退出通知**和**元数据传递**功能

在不需要子goroutine执行的时候，可以通过context通知子goroutine优雅的关闭

```Go
// context.Context
type Context interface {
   // 设置 Context 截止时间
   Deadline() (deadline time.Time, ok bool)
   // 返回一个Channel，当Context被取消或者到达截止时间时该Channel就会被关闭
   Done() <-chan struct{}
   // 返回Context结束的原因，仅在Done返回的Channel被关闭时才会返回非空值
   // 被取消返回 Canceled
   // 超时返回 DeadlineExceeded
   Err() error
   // 从Context中获取键对应的值
   Value(key interface{}) interface{}
}
```

Context 的方法都是幂等的，多次调用返回的值相同

### 语法

通过 `context.Backgroud()` 或 `context.TODO()` 进行**根 context 创建**后，可利用 `context` 包中提供的 With 系列函数来**创建功能 context 以及相应的 `CancelFunc`**
```Go
// 主动Cancel结束goroutine
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
// 绝对定时结束goroutine
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
// 相对定时结束goroutine
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
// 向子goroutine传值
func WithValue(parent Context, key, val interface{}) Context
```

当前 context 称为父 context，派生出的新 context 称为子 context，整体类似树结构

通过功能 context 即可设置子 gouroutine 的执行时间或主动控制子 goroutine 的取消
具体为当子 context 到期时或父 goroutine 主动执行 `CancelFunc` 时，子 Context 被取消，子 gouroutine 中可通过子 Context 的 `Done` 函数接收到停止信号，进而主动控制子 goroutine 关闭

### 实现

Context 包中包含canceler接口用于取消方法的实现
```Go
type canceler interface {
   cancel(removeFromParent bool, err error)  // 创建cancel接口实例的goroutine 调用cancel方法通知被创建的goroutine退出
   Done() <-chan struct{}  // 返回一个channel，后续被创建的goroutine通过监听这个channel的信号来完成退出
}
```
如果一个示例既实现了 context 接口又实现了 canceler 接口，那么这个 context 就是可以被取消的
如果仅仅只是实现了context接口，而没有实现canceler，就是不可取消的，比如emptyCtx 和valueCtx

Context 底层借助 channel 与 sync.Mutex 实现

Context 包中对 context 接口有四种基本的实现

#### `emptyCtx`

实现不具备任何功能的 context 接口，一般用它作为根 context 来派生出有实际用处的 contex
```Go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
   return
}

func (*emptyCtx) Done() <-chan struct{} {
   return nil
}

func (*emptyCtx) Err() error {
   return nil
}

func (*emptyCtx) Value(key any) any {
   return nil
}
```

#### `cancelCtx`

```Go
type cancelCtx struct {
   Context // 组合了一个Context ，所以cancelCtx 一定是context接口的一个实现
   mu   sync.Mutex // 互斥锁，用于保护以下三个字段
   done atomic.Value  // 一个chan struct{}类型，原子操作做锁优化        
   children map[canceler]struct{} // key是一个取消接口的实现
                                  // map存储当前canceler接口的子节点
                                  // 当前context被取消时遍历子节点发送取消信号
   err      error // context被取消的原因
}
```

```Go
func (c *cancelCtx) Done() <-chan struct{} {
   d := c.done.Load()
   if d != nil {
      return d.(chan struct{})
   }
   c.mu.Lock()
   defer c.mu.Unlock()
   d = c.done.Load()
   if d == nil {
      d = make(chan struct{})
      c.done.Store(d)
   }
   return d.(chan struct{})
}
```

`Done` 函数返回一个只读的 channel，在使用上要配合 select 来非阻塞读取，仅在关闭这个 channel 的时候会读到零值

```Go
// cancel closes c.done, cancels each of c's children, and, if
// removeFromParent is true, removes c from its parent's children.
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
   if err == nil {   // context被取消的原因，必传，否则panic
      panic("context: internal error: missing cancel error")
   }
   c.mu.Lock()
   if c.err != nil {  // c.err已经有值说明已经被调用过cancel函数了，当前context已经被取消
      c.mu.Unlock()
      return // already canceled
   }
   c.err = err       // 赋值err信息
   d, _ := c.done.Load().(chan struct{})  // 获取通知管道
   if d == nil {                        
      c.done.Store(closedchan)
   } else {
      close(d)                           // 关闭管道          
   }
   // 遍历当前context的所有子节点，调用取消函数
   for child := range c.children {
      // 在持有父锁的情况下获取自锁
      child.cancel(false, err)  // 递归取消子context
   }
   c.children = nil  // 取消动作完成之后，孩子节点置空
   c.mu.Unlock()

   if removeFromParent {
      removeChild(c.Context, c)  // 将自身从父节点children map种移除
}
```

`cancel` 函数不仅取消当前 context，还会递归取消当前 context 的所有子 context，之后会将自身从父节点 children map 中移除

#### `timerCtx`

`timerCtx` 在 `cancelCtx` 的基础上提供了截止时间的功能，可以设置一个截止时间 deadline ，在 deadline 到来时自动取消 context
```Go
type timerCtx struct {
   cancelCtx
   timer *time.Timer // Under cancelCtx.mu.
   deadline time.Time
}
```

父 context 未取消的情况下，在创建 timerCtx 的时候有两种情况：
- 设置的截止时间晚于父 context 的截止时间，则不会创建 timerCtx 而是直接创建一个可取消的 context，因为父 context 的截止时间更早，会先被取消，父 context 被取消的时候会级联取消这个子 context
- 设置的截止时间早于父context的截止时间，会创建一个正常的timerCtx

#### `valueCtx`

`valueCtx` 不用于父子 context 之间的取消，而是用于数据共享，作用类似于一个 map，不过数据的存储和读取分别在两个 context 上进行，用于 goroutine 之间的数据传递

```Go
type valueCtx struct {
    Context
    key, val interface{}
}
```

```Go
func (c *valueCtx) Value(key interface{}) interface{} {
   if c.key == key {
      return c.val
   }
   return c.Context.Value(key)
}
```
`Value` 函数向上递归寻找 key 所对应的 value，直到根节点返回 nil 值

### 使用场景

- 用来在 goroutine 之间传递上下文信息，比如传递请求的 trace_id，以便于追踪全局唯一请求
- 用来做取消控制，通过取消信号和超时时间来控制子 goroutine 的退出，防止 goroutine 泄漏

## Strings

Strings 是专门用于操作字符串的库
使用`strings.Builder`可以进行字符串拼接，提供了`writeString`方法拼接字符串

## Container

### List

来自 `container/list` 包，
初始化：
```Go
ls1 := list.New()
var ls2 list.List
```

方法
```Go
type List interface{
  Front()        // 返回头元素
  Back()         // 返回尾元素
  PushFront()    // 头部添加元素
  PushBack()     // 尾部添加元素
  InsertAfter()  // 在当前元素后添加元素
  InsertBefore() // 在当前元素前添加元素
  Remove()       // 删除元素
  Next()         // 返回list的下一个元素
}
```

List 用 `interface{}` 接收和返回元素，因此在用 list 元素赋值时需断言
