---
{"dg-publish":true,"permalink":"/Code/5.Go/Go.0a2.复合数据类型/","title":"复合数据类型","noteIcon":""}
---


# 复合数据类型

## `_type ` 结构体

`_type ` 是 go 里面所有类型的一个抽象，里面包含了类型的大小，哈希，对齐以及 k 类型编号等信息
Go 语言各种数据类型都是在 `_type` 字段的基础上，增加一些额外的字段来进行管理的

```go
type _type struct {
	size       uintptr // 类型大小
  ptrdata    uintptr // 前缀持有所有指针的内存大小
  hash       uint32  // 类型的hash值
  tflag      tflag   // 信息标志，和反射相关
  align      uint8   // 类型在内存中的对齐方式
  fieldAlign uint8   // 结构体字段的对齐方式
	kind       uint8   // 类型的编号，有bool, slice, struct 等等等等
	
	equal func(unsafe.Pointer, unsafe.Pointer) bool // 类型的比较函数
	
	// gc 相关
	gcdata    *byte     // 一个指向GC数据的指针
	str       nameOff   // 类型名称在字符串表中的偏移量
	ptrToThis typeOff   // 指向该类型的指针在类型表中的偏移量
}
```

```go
type interfacetype struct {
	typ     _type
	pkgpath name
	mhdr    []imethod
}

type maptype struct {
	typ    _type
	key    *_type
	elem   *_type
	bucket *_type // internal type representing a hash bucket
	// function for hashing keys (ptr to key, seed) -> hash
	hasher     func(unsafe.Pointer, uintptr) uintptr
	keysize    uint8  // size of key slot
	elemsize   uint8  // size of elem slot
	bucketsize uint16 // size of bucket
	flags      uint32
}

type arraytype struct {
	typ   _type
	elem  *_type
	slice *_type
	len   uintptr
}

type chantype struct {
	typ  _type
	elem *_type
	dir  uintptr
}

type slicetype struct {
	typ  _type
	elem *_type
}

type functype struct {
	typ      _type
	inCount  uint16
	outCount uint16
}

type ptrtype struct {
	typ  _type
	elem *_type
}

type structfield struct {
	name   name
	typ    *_type
	offset uintptr
}

type structtype struct {
	typ     _type
	pkgPath name
	fields  []structfield
}
```

## [[Code/5.Go/GO.0a2a.Slice\|Slice]]

## [[Code/5.Go/Go.0a2b.Map\|Map]]

## String

> [Strings, bytes, runes and characters in Go](https://go.dev/blog/strings)

String 类型对应一个 struct
该 struct 包含两个元素，第一个是指向该 string 的字节数组的指针，第二个是 string 的长度，每个元素占 8 个字节
```Go
type StringHeader struct {
    Data uintptr // 指向字节数组的指针
    Len  int     // string的长度
}
```

- String 初始值为空字符串 `""`，但 `"" != nil
- String 指向字符串字面量，不允许修改，但可以通过下标遍历
- String 的赋值操作仅为结构体复制，不会涉及底层字节数组的复制
- String 是 8bit 字节的集合，内部实现使用 UTF-8 编码
- String 拼接实际上创建了一个新字符串并进行内存拷贝
- 以 range 遍历 string 返回 rune 类型，以下标遍历返回 byte 类型

>[!Notice] string 和 []byte 的相互拷贝
string 和 `[]byte` 类型互转通常都通过**内存拷贝**的形式完成
但在例如 `string(bytes) == "content"` 等临时转换而非赋值的情况下，不发生内存拷贝，而是生成一个临时 string，其内指针指向字符数组

### 声明

```Go
str1 := "Hello World"
str2 := `Hello
Golang`
```

反引号声明的 string 中的特殊字符不需转义，所见即所得

## Struct

```Go
// 使用new初始化
st := new(structName)
// 键值对初始化
st := structName{
  args1: value1,
  args2: value2,
  ...
}  
// 值列表初始化
st := stuctName{
  value1,
  value2,
  ...
}
```

无论 struct 指针还是 struct 变量，访问 struct 成员都使用 `a.arg`，实际上 Go 语言内部编译器将指针 `a` 转换为 `*a`

### 嵌套

> [第十八章 Struct 结构体 · Go 语言 42 章经 (gitbooks.io)](https://wizardforcel.gitbooks.io/go42/content/content/42_18_struct.html)

Struct 中的字段可以不用给名称，这时称为匿名字段，**匿名字段的名称强制和类型相同**

通过在 struct 内加入匿名字段可以以隐式继承该字段的方法及属性
```Go
type A

func (a *A) methodA()

type B struct {
	A
}

B.methodA() // 编译器自动解释为B.A.methodA()
```

### Empty struct

**空结构体不占据内存空间**，因此可以用于占位符节省资源

- 可以通过 `map[int]struct{}` 方式实现集合
- 使用空结构体作为 channel 占位符
- 空结构体构建只包含方法不占用空间的结构体

> [Go 空结构体 struct{} 的使用 | Go 语言高性能编程 | 极客兔兔 (geektutu.com)](https://geektutu.com/post/hpg-empty-struct.html)

### Tag

Go 标签在结构体中使用，在结构体编译阶段，可以通过反射被获取。

`Tag` 本身是一个字符串，它是 **以空格分隔的 key:value 对，**
- key : 必须是非空字符串，不能包含控制字符、空格、引号、冒号
- value : 以双引号标记的字符串
- 注意 ：冒号前后不能有空格，键值对之间要以空格间隔
```go
struet demo {
    demo int `k1:"v1" k2:"v2"`
}
```

在 Go 中，tag 有着方便 json 字符串以及常用框架解析的重要作用

```go
type Student struct { 
    ID int `json:"-"` // 该字段不进行序列化 
    Class int `json:"class"` // 该字段与class对应
    Name string `json:"name,omitempy"` // 如果为类型零值或空值，序列化时忽略该字段
    Age int `json:"age,string"` // 指定类型，支持string、number、boolen 
}
```

[[Code/5.Go/Go.0a2c.Interface\|Go.0a2c.Interface]]

[[Code/5.Go/Go.0a2d.Channel\|Go.0a2d.Channel]]