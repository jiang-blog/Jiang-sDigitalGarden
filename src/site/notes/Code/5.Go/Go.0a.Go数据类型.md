---
{"dg-publish":true,"permalink":"/Code/5.Go/Go.0a.Go数据类型/","title":"Go 数据类型","noteIcon":""}
---


# Go 数据类型

## 数据属性

### Const 常量

- **只能定义基本类型的常量，不能定义切片，数组，指针，结构体等这些类型的常量**
    - 布尔类型 `bool`，无符号整数 `uint/uint8/uint16/uint32/uint64/uintptr`，有符号整数 `int/int8/int16/int32/int64`，浮点数 `float32/float64`，或者底层类型是这些基本类型的类型(e.g. `time.Duration`)
- **指针不能指向常量**
- 常量间的所有算术运算、逻辑运算和比较运算的结果也是常量，对常量的类型转换操作或以下函数调用都是返回常量结果：`len`、`cap`、`real`、`imag`、`complex` 和 `unsafe.Sizeof`
- 对于批量声明的常量，除了第一个外其它常量右边的初始化表达式都可以省略，表示使用前面常量的初始化表达式写法，对应的常量类型也一样

```go
const (
    a = 1
    b
    c = 2
    d
)

fmt.Println(a, b, c, d) // "1 1 2 2"
```

#### 无类型常量

一个常量的声明可以包含一个类型和一个值，但是如果没有显式指明类型，那么将从右边的表达式推断类型

虽然一个常量可以有任意一个确定的基础类型，但是如上文中 `a,b,c,d` 常量并没有一个明确的基础类型

编译器为这些没有明确的基础类型的数字常量提供比基础类型更高精度的算术运算，可以认为至少有 256bit 的运算精度
有六种未明确类型的常量类型，分别是无类型的布尔型、无类型的整数、无类型的字符、无类型的浮点数、无类型的复数、无类型的字符串

通过延迟明确常量的具体类型，不仅可以提供更高的运算精度，而且可以直接用于更多的表达式而不需要显式的类型转换

无类型整数常量转换为 int 的内存大小是不确定的，但是无类型浮点数和复数常量则转换为内存大小明确的 float64 和 complex128

#### Iota

**iota**可认为是 const 语句块中的行索引
```Go
package main

import "fmt"

func main() {
	const (
		a = iota   //0
		b          //1
		c          //2
		d = "ha"   //独立值
		e          //"ha"
		f = 100    //
		g          //100
		h = 1+iota //8，当前为第7行,iota=7
		i          //9
	)
	fmt.Println(a,b,c,d,e,f,g,h,i)
}
```
结果为
```Go
0 1 2 ha ha 100 100 8 9
```

#### enum

go 没有提供 enum 枚举类型，[可以使用 const 来模拟](https://www.jianshu.com/p/ce95d7443c97)
```go
type PolicyType int32

const (
    Policy_MIN      PolicyType = 0
    Policy_MAX      PolicyType = 1
    Policy_MID      PolicyType = 2
    Policy_AVG      PolicyType = 3
)

func (p PolicyType) String() string {
    switch (p) {
    case Policy_MIN: return "MIN"
    case Policy_MAX: return "MAX"
    case Policy_MID: return "MID"
    case Policy_AVG: return "AVG"
    default:         return "UNKNOWN"
    }
}

func foo(p PolicyType) {
    fmt.Printf("enum value: %v\n", p)
}

func main() {
    foo(Policy_MAX) // 输出 enum value: MAX
}
```

### 变量声明 var

Go 通过 `var {varible_name} {type}` 形式定义变量

局部变量定义但不使用会产生编译错误，但全局变量定义不使用不产生错误
**全局变量不能使用 `:=` 定义**，必须以 var 关键字开头

变量通过首字母大小写方式区分 public 和 private，variable/method/struct 的首字母大写时包外可见

### 类型别名 & 类型定义

定义类型别名

```go
type TypeAlias = Type
```

定义类型

```go
type NewType Type
```

## 输出格式化

[Go 之 格式化输出 《Go语言基础》 - 掘金 (juejin.cn)](https://juejin.cn/post/7042636455239221256)

| 格式      | 说明                                                   |
| --------- | ------------------------------------------------------ |
| %d        | 格式化整数                                             |
| %+d       | 输出数值的符号                                         |
| %(-){n}d  | 用于规定输出定长 n 的整数，默认右对齐，-号代表左对齐   |
| %e        | 科学计数表示法                                         |
| %f        | 格式化浮点数                                           |
| %g        | 根据浮点数的大小自动选择使用 `%f` 或 `%e` 来输出浮点数 |
| %{n}.{m}g | 用于表示宽度 n 并精确到小数点后 m 位                   |
| %b        | 格式化 2 进制                                          |
| %o        | 格式化 8 进制                                          |
| %X, %x    | 格式化 16 进制表示的数字，X-字母大写                   |
| %U, %u    | Unicode，格式为 U+hhhh 的字符串                        |
| %t        | 格式化布尔型                                           |
| %c        | 格式化字符                                             |
| %s        | 格式化字符串                                           |
| %p        | 格式化指针                                             |
| %v        | 使用类型的默认输出格式的标识符                         |

调试相关格式化说明符

| 格式 | 说明                                           |
| ---- | ---------------------------------------------- |
| %T   | 打印某个类型的完整说明                         |
| %+v  | 打印包括字段在内的实例的完整信息               |
| %#v  | 打印包括字段和限定类型名称在内的实例的完整信息 |

### 字符串宽度控制

最小宽度, 不够部分可以选择补 0
```go
fmt.Printf("|%s|", "aa")   // |aa|    不设置宽度
fmt.Printf("|%5s|", "aa")  // |   aa| 5个宽度,  默认右对齐
fmt.Printf("|%-5s|", "aa") // |aa   | 5个宽度, 左对齐
fmt.Printf("|%05s|", "aa") // |000aa| 5个宽度,用0补齐
```

最大宽度, 超出的部分会被截断
```go
fmt.Printf("|%.5s|", "xxxxxxx") // |xxxxx| 最大宽度为5
```

不同语言的文字宽度并不一定相同, 比如

```go
fmt.Printf("|%2s|", "中国")  // |中国|
fmt.Printf("|%2s|", "ab")   // |ab|
```

### 浮点数精度控制

```go
a := 54.123456
fmt.Printf("|%f|", a)     // |54.123456|
fmt.Printf("|%+f|", a)    // |+54.123456|
fmt.Printf("|%5.1f|", a)  // | 54.1|
fmt.Printf("|%-5.1f|", a) // |54.1 |
fmt.Printf("|%05.1f|", a) // |054.1|
```

### 格式化错误

类型错误或未知: `%!verb(type=value)`
```go
fmt.Printf("%d", "hi") // %!d(string=hi)
```


太多参数: `%!(EXTRA type=value)`
```go
fmt.Printf("hi", "guys") // hi%!(EXTRA string=guys)
```


太少参数: `%!verb(MISSING)`
```go
fmt.Printf("hi%d") // hi %!d(MISSING)
```

宽度/精度不是整数值: `%!(BADWIDTH) or %!(BADPREC)`
```go
fmt.Printf("%d", hi) // %!d(string=hi)
```

## 类型比较

> [Golang 之 struct能不能比较 - 掘金 (juejin.cn)](https://juejin.cn/post/6881912621616857102)

Go 语言数据类型比较：
- 可直接比较：_Integer_，_Floating-point_，_String_，_Boolean_，_Complex(复数型)_，_Pointer_，_Channel_，_Interface_，_Array_，*Struct*
- 不可直接比较：_Slice_，_Map_，_Function_

成员皆可直接比较时同类型的 struct 和 array 可比较
**struct 必须是可比较的，才能作为 map 的 key**

不可直接比较的数据类型可通过 `reflect.DeepEqual` 进行比较，比较规则如下：
- 不同类型的值永远不会深度相等
- 当两个数组的对应元素深度相等时，两个数组深度相等
- 当两个相同结构体的所有字段对应深度相等的时候，两个结构体深度相等
- 当两个函数都为 nil 时，两个函数深度相等，其他情况不相等(相同函数也不相等)
- 当两个 interface 的真实值深度相等时，两个 interface 深度相等
- Map 的比较需要同时满足以下几个
    - 两个 map 都为 nil 或者都不为 nil 且长度相等
    - 相同的 map 对象或者所有 key 要对应相同
    - Map 对应的 value 也要深度相等
- 指针，满足以下其一即是深度相等
    - 两个指针满足 go 的 `==` 操作符
    - 两个指针指向的值是深度相等的
- Slice，需要同时满足以下几点才是深度相等
    - 两个切片都为 nil 或者都不为 nil，并且长度要相等
    - 两个切片底层数据指向的第一个位置要相同或者底层的元素要深度相等
    - 注意：空的切片跟 nil 切片是不深度相等的
- 其他类型的值(numbers, bools, strings, channels)如果满足 go 的 `==` 操作符，则是深度相等的。要注意不是所有的值都深度相等于自己，例如函数，以及嵌套包含这些值的结构体，数组等

## 类型转换

> [golang 中的四种类型转换总结 | Go 技术论坛 (learnku.com)](https://learnku.com/articles/42797)

go 存在 4 种类型转换，分别为：断言、强制、显式、隐式

断言
`var s = x.(T)`
强制：
```go
var f float64
bits = *(*uint64)(unsafe.Pointer(&f))
```
显式：
```go
int64(222)
[]byte("ssss")
```
隐式：
```go
type Handler func()

func main() {
    var i interface{} = main
    _, ok := i.(func()) 
    fmt.Println(ok)     // true
    _, ok = i.(Handler) 
    fmt.Println(ok)     // false
    fmt.Println(reflect.TypeOf(main) == reflect.TypeOf((*Handler)(nil)).Elem()) // false
}
```
