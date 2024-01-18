---
{"dg-publish":true,"permalink":"/Code/5.Go/Go.0a2c.Interface/","title":"Interface","noteIcon":""}
---


# Interface

> [go interface 设计与实现 - 掘金 (juejin.cn)](https://juejin.cn/post/7173965896656879630)

接口是一组方法的声明的集合，依托于自定义类型对方法的具体实现而存在
**实现了接口中所有方法的类型就可以为该接口的实例赋值**，但一个接口的实例仅可调用接口中的方法

```Go
type interfaceName interface {
   methodName1([parameter_list]) [return_type_list]
   methodName2([parameter_list]) [return_type_list]
   methodName3([parameter_list]) [return_type_list]
   ...
}
```
interface 同样可作为匿名字段嵌入 struct，超集接口对象可转换为子集接口对象，反之出错

接口的零值是指*接口类型*和*接口值*都为 `nil`
当仅且当 interface 的**动态类型**和值都为 `nil` 的情况下， `itfA == nil` 才为 `true`
> [接口的动态类型和动态值 | Go 程序员面试笔试宝典 (golang.design)](https://golang.design/go-questions/interface/dynamic-typing/)

```go
type Coder interface {
	code()
}
type Gopher struct {
	name string
}
func (g Gopher) code() {
	fmt.Printf("%s is coding\n", g.name)
}

func main() {
	var c Coder
	fmt.Println(c == nil)            // true
	fmt.Printf("c: %T, %v\n", c, c)  // c: <nil>, <nil>

	var g *Gopher
	fmt.Println(g == nil)            // true

	c = g
	fmt.Println(c == nil)            // false
	fmt.Printf("c: %T, %v\n", c, c)  // c: *main.Gopher, <nil>
}
```

## interface 使用

### 检测接口实现

```go
var _ interfaceName = (*structName)(nil)
var _ interfaceName = structName{}
```

通过该语句，编译器检测某一类型是否实现了某个接口

### 断言

使用接口为其他类型变量赋值时需使用**断言**
断言通过以下表达式判断 `element` 变量是否为类型 `T`，是则返回类型为 T 的对象值
```Go
value := element.(T) // 非安全断言，失败引发 panic
value, ok := element.(T) // 安全断言，失败时 ok 为 false
```

此外可以利用 `switch` 语句判断接口的类型

```go
switch v := v.(type) {
	case nil:
		fmt.Printf("%p %v\n", &v, v)
		fmt.Printf("nil type[%T] %v\n", v, v)
	case Student:
		fmt.Printf("%p %v\n", &v, v)
		fmt.Printf("Student type[%T] %v\n", v, v)
	case *Student:
		fmt.Printf("%p %v\n", &v, v)
		fmt.Printf("*Student type[%T] %v\n", v, v)
	default:
		fmt.Printf("%p %v\n", &v, v)
		fmt.Printf("unknow\n")
}
```

`switch…case` 或 `v, ok := i.(T)` 允许断言失败，而 `i.(T).xx()` 或 `v := i.(T)` 不允许断言失败，否则 panic

### 类型集

>  [泛型 | 李文周的博客 (liwenzhou.com)](https://www.liwenzhou.com/posts/Go/generics/)

**V1.18开始接口类型的定义也发生了改变，由过去的接口类型定义方法集（method set）变成了接口类型定义类型集（type set）** 
也就是说，接口类型现在既可以用作值的类型，又可以用作类型约束
> ![|825](https://www.liwenzhou.com/images/Go/generics/type-set.png)

类型集相较于方法集的优势在于可以显式地向集合添加类型，从而以新的方式控制类型集
```go
type V interface {
	int | string
	~bool
}
```

`T1 | T2` 表示类型约束为 T1和 T2这两个类型的并集
`~T` 表示所有底层类型是 T 的类型，例如 `~string` 表示所有底层类型是 `string` 的类型集合
### 实现多态

## interface 实现

### 非空接口

```go
type iface struct {
   tab  *itab          // 方法表
   data unsafe.Pointer // 指向变量本身的指针
}

// itab结构里面存储接口要求的方法列表和data对应的动态类型信息
type itab struct {
   inter *interfacetype // 指向接口类型的描述信息
   _type *_type         // 指向实际类型的描述信息
   hash  uint32         // 该类型的hash值，用于快速判断类型是否相等，能否转换
   _     [4]byte
   fun   [1]uintptr     // 一个指针数组，保存了变量的接口方法地址，以在调用的时候快速定位
                        // 如果该接口对应的动态类型没有实现接口的所有方法，fun[0]=0，表示断言失败，该类型不能赋值给该接口
}

// 记录接口类型的描述信息，主要是接口的方法列表
type interfacetype struct {
   typ     _type      // 类型信息
   pkgpath name       // 包路径
   mhdr    []imethod  // 接口的方法列表
}
```

go 底层的类型信息使用 `_type` 结构体存储

- `itab` 实际上定义了接口类型和变量实际类型之间方法的交集
- `itab` 包含了接口的所有方法，这些方法是实际类型的子集
- `itab` 里面的方法列表包含了实际类型的方法指针(也就是实际类型的方法的地址)，通过这个地址可以对实际类型进行方法的调用
- `itab` 在实际类型没有实现接口的所有方法的时候，生成失败(生成的 `itab` 里面的方法列表是空的，`fun[0] = 0` )

判断某一个结构体是否实现了某一个接口时，通过比较两者的方法集生成 `itab`
go 中定义了一个全局变量 `itabTable` 用来缓存 `itab`
```go
// 全局的 itab 表
itabTable     = &itabTableInit
itabTableInit = itabTableType{size: itabInitSize}

// 表里面缓存了 itab
type itabTableType struct {
    size    uintptr             // entries 的长度，2 的次方
    count   uintptr             // 当前 entries 的数量
    entries [itabInitSize]*itab // 保存 itab 的哈希表
}
```
entries 用 `interfacetype` (接口类型信息)和 `_type` (实际类型信息)生成一个哈希表的键

接口之间相互转换时，使用 `getitab` 函数根据 `interfacetype` 和 `_type ` 去全局的 itab 哈希表中查找，如果能找到，则直接返回；否则，会根据给定的 `interfacetype` 和 `_type` 新生成一个 itab，并插入到 itab 哈希表，这样下一次就可以直接拿到 itab

###  空接口

没有任何方法声明的接口为空接口 `interface{}`，因此所有类型都实现了空接口
空接口是任意对象的子集，可以将任意类型的数据赋值给一个空接口

```go
type eface struct {
   _type *_type         // 指向实际类型的描述信息
   data  unsafe.Pointer // 指向变量本身的指针
}
```

### 接口断言

**go 的接口断言本质上是类型转换**
