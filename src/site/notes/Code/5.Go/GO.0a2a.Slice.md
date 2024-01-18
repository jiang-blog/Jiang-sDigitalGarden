---
{"dg-publish":true,"permalink":"/Code/5.Go/GO.0a2a.Slice/","title":"Slice","noteIcon":""}
---


# Slice

## Array

数组为具有**固定长度**的基本数据结构，长度大于元素个数的时候用零值补位，*不能用 make 初始化，不能动态设定长度*
array 初始化：
```Go
var arr = [constant]int{a,b,c}
var arr = [...]int{a,b,c}       // 数组长度由初始化元素个数决定
```

**数组的长度也是数组类型的组成部分**，不同长度的数组不能用 `==` 进行比较

## Slice

slice 本质上为对数组的**引用**，对 slice 的改变等于直接改变数组

用 make 函数创建初始化 slice 时会返回一个 `reflect.SliceHeader` 结构体
```Go
type SliceHeader struct {
	Data uintptr // 底层数组指针
	Len  int     // slice长度
	Cap  int     // slice容量
}
```

slice 初始化：
```Go
// len(slice1)=high-low cap(slice1)=len(array)-low
slice1 := array[low:high:max]             // max 不能超过原数组长度
slice2 := make([]type, length, capacity)  // 等同于 make([]T, cap)[:len]
slice3 := []int{a,b,c}
```

当使用 `append` 对 slice 进行追加时，如果超出原数组大小，go 会拷贝创建一个更大的新数组，令 slice 变为新数组的引用
```Go
// "..."将slice2打散为元素加入slice1
slice1 = append(slice1,slice2...)
```

3 种特殊 slice 类型：
- 0-slice：slice 元素未赋值，都是类型 0 值
- empty-slice：底层数组指针为特殊指针 `zerobase`，empty slice 的 `len` 为 0，`cap` 可以为任意值
- nil-slice：未初始化 slice，底层数组指针为 nil，nil slice 的 `len` 和 `cap` 都为 0

```Go
var s []string             // nil slice    len(s) == 0  s == nil
var s = make([]string, 0)  // empty slice
var s = make([]string, 5)  // 0 slice
```

```go
func hello(num ...int) {  
    num[0] = 18
}
func main() {  
    i := []int{5, 6, 7}
    hello(i...)
    fmt.Println(i[0])   // 18
}
```

### 复制 - copy

对 slice 进行等号复制为指针复制，改变原切片或新切片都会对另一个产生影响
slice 作为参数传递给函数时进行浅拷贝，函数对 slice 的操作实际为对底层数组的操作

copy 复制为值复制，改变原切片的值不会影响新切片，copy 复制会比等号复制慢
```go
slice1 := []int{1, 2, 3, 4, 5}
slice2 := []int{5, 4, 3}
copy(slice2, slice1) // 只会复制slice1的前3个元素到slice2中
copy(slice1, slice2) // 只会复制slice2的3个元素到slice1的前3个位置
```

### 扩容 - append

1. 初步扩容
    - V1.18 以前：
      - 如果新长度 `newLen` 大于原容量的两倍，将新容量设置为 `newLen`
      - 如果原容量小于 `1024` ，新容量为两倍原容量
      - 如果原容量不小于 `1024`，新 slice 容量变成原来的 `1.25` 倍
    - [V1.18以后](https://github.com/golang/go/blob/581603cb7d02019bbf4ff508014038f3120a3dcb/src/runtime/slice.go#L166)：
      - 如果新长度 `newLen` 大于原容量的两倍，将新容量设置为 `newLen`
      - 如果原容量小于 `threshold` (256)，新容量为两倍原容量
      - 如果原容量不小于 `threshold` (256)，将新容量按照公式 `newCap = oldCap+(oldCap+3*256)/4` 循环增加，使得新容量最终不小于 `newLen`
2. 根据元素类型的大小微调新容量(只增不减)，进行内存对齐

确认容量后，向 Go 内存管理器申请内存，将老 slice 中的数据复制过去，并且将 append 的元素添加到新的底层数组中

```go
s := []int{5}
s = append(s, 7)
s = append(s, 9)
x := append(s, 11)
y := append(s, 12)
fmt.Println(s, x, y)
// [5 7 9] [5 7 9 12] [5 7 9 12]
```

**`nil slice` 或 `empty slice` 都可以通过调用 append 函数来获得底层数组的扩容**
最终都是调用 `mallocgc` 向 Go 的内存管理器申请到一块内存，然后再赋给原来的 `nil slice` 或 `empty slice`，使其成为真正的 slice

### 传参

slice 作为引用类型，通过传入参数可以改变其底层数据，但不能改变其结构本身(长度，容量)
