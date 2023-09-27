---
{"dg-publish":true,"permalink":"/Code/5.Go/Go.2.Go标准库/","title":"Go 标准库","noteIcon":""}
---


# Go 标准库

## Runtime

runtime作为程序的一部分打包进二进制产物，随用户程序一起运行，跟用户程序没有明显界限，它提供一些go运行基础功能，gc，gmp，内存管理等，常用的go，make等关键词都是runtime提供的

### runtime.SetFinalizer

go提供了`runtime.SetFinalizer`函数，当GC准备释放对象时，会回调该函数指定的方法

[[Code/5.Go/Go.2b.Sync\|Go.2b.Sync]]
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

## JSON

> [Go 语言 JSON 的实现原理 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part4-advanced/ch09-stdlib/golang-json/)

Go 通过 [`encoding/json`](https://golang.org/pkg/encoding/json/) 对外提供标准的 JSON 序列化和反序列化方法，即 `encoding/json.Marshal` 和 `encoding/json.Unmarshal`
### 接口

JSON 标准库中提供了 [`encoding/json.Marshaler`](https://draveness.me/golang/tree/encoding/json.Marshaler) 和 [`encoding/json.Unmarshaler`](https://draveness.me/golang/tree/encoding/json.Unmarshaler) 两个接口分别可以影响 JSON 的序列化和反序列化结果：

```go
type Marshaler interface {
	MarshalJSON() ([]byte, error)
}

type Unmarshaler interface {
	UnmarshalJSON([]byte) error
}
```

在 JSON 序列化和反序列化的过程中，它会使用反射判断结构体类型是否实现了上述接口，如果实现了上述接口就会优先使用对应的方法进行编码和解码操作，除了这两个方法之外，Go 语言其实还提供了另外两个用于控制编解码结果的方法，即 [`encoding.TextMarshaler`](https://draveness.me/golang/tree/encoding.TextMarshaler) 和 [`encoding.TextUnmarshaler`](https://draveness.me/golang/tree/encoding.TextUnmarshaler)：

```go
type TextMarshaler interface {
	MarshalText() (text []byte, err error)
}

type TextUnmarshaler interface {
	UnmarshalText(text []byte) error
}
```

一旦发现 JSON 相关的序列化方法没有被实现，上述两个方法会作为候选方法被 JSON 标准库调用并参与编解码的过程

总的来说，我们可以在任意类型上实现上述这四个方法自定义最终的结果，后面的两个方法的适用范围更广，但是不会被 JSON 标准库优先调用

### 标签

Go 语言的字段一般都是驼峰命名法，JSON 中下划线的命名方式相对比较常见，所以通常使用标签这一特性直接建立键与字段之间的映射关系

JSON 中的标签由标签名和标签选项两部分组成，标签名和字段名会建立一一对应的关系，后面的标签选项也会影响编解码的过程
标签名和标签选项都以 `,` 连接，最前面的字符串为标签名，后面的都是标签选项
如下所示的 `name` 和 `age` 是标签名
```go
type Author struct {
    Name string `json:"name,omitempty"`
    Age  int32  `json:"age,string,omitempty"`
}
```

常见的两个标签选项是 `string` 和 `omitempty`
`string`  表示当前的整数或者浮点数是由 JSON 中的字符串表示的
`omitempty` 会在字段为零值时在生成的 JSON 中忽略对应的键值对，例如：`"age": 0 `、`"author": ""` 等

### 其他

json 标准库在遇到管道/函数等无法被序列化的内容时会发生错误

## HTTP

HTTP 库实现了 HTTP/1.1 和 HTTP/2.0 两大版本的 http 协议

Go 语言的 `net/http` 中同时包好了 HTTP 客户端和服务端的实现，为了支持更好的扩展性，它引入了 `net/http.RoundTripper` 和 `net/http.Handler` 两个接口

`net/http.RoundTripper` 是用来表示执行 HTTP 请求的接口，调用方将请求作为参数可以获取请求对应的响应
```go
type RoundTripper interface {
    RoundTrip(*Request) (*Response, error)
}
```

`net/http.Handler` 主要用于 HTTP 服务器响应客户端的请求，HTTP 请求的接收方可以实现 `net/http.Handler` 接口，其中实现了处理 HTTP 请求的逻辑，处理的过程中会调用 `net/http.ResponseWriter` 接口的方法构造 HTTP 响应，它提供的三个接口 `Header`、`Write` 和 `WriteHeader` 分别会获取 HTTP 响应、将数据写入负载以及写入响应头
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

type ResponseWriter interface {
	Header() Header
	Write([]byte) (int, error)
	WriteHeader(statusCode int)
}
```

客户端和服务端面对的都是双向的 HTTP 请求与响应，客户端构建请求并等待响应，服务端处理请求并返回响应

### 客户端

客户端可以直接通过 `net/http.Get` 使用默认的客户端 `net/http.DefaultClient` 发起 HTTP 请求，也可以自己构建新的 `net/http.Client` 实现自定义的 HTTP 事务
在多数情况下使用默认的客户端都能满足需求，不过使用默认客户端发出的请求没有超时时间，所以在某些场景下会一直等待下去
除了自定义 HTTP 事务之外，还可以实现自定义的 `net/http.CookieJar` 接口管理和使用 HTTP 请求中的 Cookie
