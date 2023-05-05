---
{"dg-publish":true,"permalink":"/Code/6.Go/Go.0b.Go标准库/","title":"Go 标准库","noteIcon":""}
---


# Go 标准库

## Strings

Strings 是专门用于操作字符串的库
使用`strings.Builder`可以进行字符串拼接，提供了`writeString`方法拼接字符串

## Sync

### Sync.Map

原生 map 和 slice 非线程安全

Sync.Map 为并发安全的哈希表
```Go
// https://github.com/golang/go/blob/8edcdddb23c6d3f786b465c43b49e8d9a0015082/src/sync/map.go#L35

type Map struct {
 mu Mutex
 read atomic.Pointer[readOnly]
 dirty map[interface{}]*entry
 misses int
}

// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
	m       map[interface{}]*entry
	amended bool // true if the dirty map contains some key not in m.
}

// An entry is a slot in the map corresponding to a particular key.
type entry struct {
	p atomic.Pointer[interface{}]
}
```

Sync.Map 底层**使用了两个原生 map 进行读写分离**，读数据存在只读哈希表 read 上，最新写入的数据则存在 dirty 哈希表上

优点：通过读写分离降低锁时间来提高效率
缺点：不适用于大量写的场景

#### 读取

1. 首先通过原子操作查询 read
2. 不存在时加锁再次查询 read
3. 依然不存在时查询 dirty

通过 `misses` 字段统计穿透 read 读取 dirty 的次数，超过阈值(`len(dirty)`)则**使用 dirty 数据覆盖 read**，之后将 dirty 设为 nil

#### 插入

1. 尝试直接通过原子操作更新 read 中的 key
2. 更新失败，加锁继续查询
    - Read 中存在该 key 且对应 entry 被标记为 `expunged` ，表示 dirty 中不包含该 key，添加
    - Dirty 中存在该 key，更新
    - 都不存在，添加至 dirty
        - 若 dirty 中不包含 read 中没有的数据(`read.amended == true` )，触发一次 dirty 刷新，dirty **遍历复制** read 中**未被删除的元素**

#### 删除

删除时的查找流程与读取相同
- 对 read 采用*延迟删除*，仅将 read 中目标 key 对应的 entry 值置为 nil
- 对 dirty 直接删除

Read 中已经被删除 key 对应的 entry 会先被标记为 nil，当 dirty 刷新时更新为 `expunged`，不被刷新至 dirty 中，下一次 dirty 覆盖 read 时被回收

### Sync.Mutex

> [Go精妙的互斥锁设计 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/343563412)

互斥锁
加锁：`sync.Mutex.Lock`
解锁：`sync.Mutex.Unlock`

**加锁**
1. 如果锁是完全空闲状态，则通过 CAS 直接加锁
2. 如果锁处于正常模式，则会尝试自旋，通过持有 CPU 等待锁的释放
3. 如果当前 goroutine 不再满足自旋条件，则会计算锁的期望状态，并尝试更新锁状态
4. 在更新锁状态成功后，会判断当前 goroutine 是否能获取到锁，能获取锁则直接退出
5. 当前 goroutine 不能获取到锁时，则会由 sleep 原语 `SemacquireMutex` 陷入睡眠，等待解锁的 goroutine 发出信号进行唤醒
6. 唤醒之后的 goroutine 发现锁处于饥饿模式，则能直接拿到锁，否则重置自旋迭代次数并标记唤醒位，重新进入步骤2中

**解锁**
1. 如果通过原子操作 `AddInt32` 后，锁变为完全空闲状态，则直接解锁
2. 如果解锁一个没有上锁的锁，则直接抛出异常
3. 如果锁处于正常模式，且没有 goroutine 等待锁释放，或者锁被其他 goroutine 设置为了锁定状态、唤醒状态、饥饿模式中的任一种（非空闲状态），则会直接退出；否则，会通过 wakeup 原语 `Semrelease` 唤醒 waiter。
4. 如果锁处于饥饿模式，会直接将锁的所有权交给等待队列队头 waiter，唤醒的 waiter 会负责设置 `Locked` 标志位

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
但刚被唤起的 Goroutine 与新创建的 Goroutine 竞争时大概率获取不到锁
为了减少这种情况的出现，一旦 Goroutine 超过 1ms 没有获取到锁，它就会将当前互斥锁切换为饥饿模式，防止部分 Goroutine 被饿死

**在饥饿模式中，互斥锁会直接交给等待队列最前面的 Goroutine**，新的 Goroutine 在该状态下不能获取锁，也不会进入自旋状态，只会在队列的末尾等待
如果一个 Goroutine 获得了互斥锁并且它在队列的末尾或者它等待的时间少于 1ms，那么当前的互斥锁就会被切换回正常模式

相比于饥饿模式，正常模式下的互斥锁能够提供更好的性能，饥饿模式能避免由于 goroutine 陷入等待无法获取锁而造成的高尾延时

#### 自旋

Goroutine 能进入自旋的条件如下
- 当前互斥锁处于正常模式
- 当前运行的机器是多核 CPU，且 GOMAXPROCS>1
- 至少存在一个其他正在运行的处理器 P，并且它的本地运行队列(local runq)为空
- 当前 goroutine 进行自旋的次数小于4

### Sync.RWMutex

读写锁，可同时读，不可同时写，读写不共存

### Sync.Once

确保在并发程序中一个函数仅执行一次

Once 常常用来初始化单例资源，或者并发访问只需初始化一次的共享资源，或者在测试的时候初始化一次测试资源
sync.Once 只暴露了一个幂等方法 Do，`sync.Once.Do(f())` 接收一个无参数无返回值的函数 f 作为输入，该函数在且仅在第一次调用 Do 时执行

Once 与 init 方法的不同在于 init 是在其所在的 package 首次加载时执行的，而 sync.Once 可以在代码的任意位置初始化和调用，是在第一次用的它的时候才会初始化

### Sync.WaitGroup

Go 中使用 sync.WaitGroup 来实现并发任务的同步以及协程任务等待

```Go
type WaitGroup interface{
  func (wg * WaitGroup) Add(delta int)       // 计数器加delta
  func (wg *WaitGroup) Done()                // 计数器减1
  func (wg *WaitGroup) Wait()                // 会阻塞代码的运行，直至计数器减为0。
}
```

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
如果一个示例既实现了 context 接口又实现了 canceler 接口，那么这个 context 就是可以本取消的
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
      // NOTE: acquiring the child's lock while holding parent's lock.
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
