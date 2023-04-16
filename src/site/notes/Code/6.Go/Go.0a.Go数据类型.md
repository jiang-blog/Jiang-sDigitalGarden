---
{"dg-publish":true,"permalink":"/Code/6.Go/Go.0a.Go数据类型/","title":"Go 基础数据类型","noteIcon":""}
---


# Go 数据类型

## 数据属性

### Const

#### Iota

**iota**可认为是const语句块中的行索引
```Go
package main

import "fmt"

func main() {
	const (
		a = iota   //0
		b          //1
		c          //2
		d = "ha"   //独立值，iota += 1
		e          //"ha"   iota += 1
		f = 100    //iota +=1
		g          //100  iota +=1
		h = 1+iota //8,恢复计数
		i          //9
	)
	fmt.Println(a,b,c,d,e,f,g,h,i)
}
```
结果为
```Go
0 1 2 ha ha 100 100 8 9
```

### Var

全局变量不能使用`:=`定义

如果定义了一个局部变量但不使用编译报错，但全局变量定义不使用不报错

无论是method、const、var、interface还是struct等的名称，如果首字母大写，则可以被其他的包访问；如果首字母小写，则只能在本包中使用。

### Nil

Nil 代表空指针

**nil可以作为任意指针形参的实参传入**
**nil可以调用所有指针方法**

## 基础数据类型

| 类型          | 名称     | 长度 | 零值  | 说明                                             |
| ------------- | -------- | ---- | ----- | ------------------------------------------------ |
| bool          | 布尔类型 | 1    | false | 其值不为真即为家，不可以用数字代表 true 或 false |
| byte          | 字节型   | 1    | 0     | uint8别名                                        |
| rune          | 字符类型 | 4    | 0     | 专用于存储unicode编码，等价于uint32              |
| int, uint     | 整型     | 4或8 | 0     | 32位或64位                                       |
| int8, uint8   | 整型     | 1    | 0     | -128 ~ 127, 0 ~ 255                              |
| int16, uint16 | 整型     | 2    | 0     | -32768 ~ 32767, 0 ~ 65535                        |
| int32, uint32 | 整型     | 4    | 0     | -21亿 ~ 21 亿, 0 ~ 42 亿                         |
| int64, uint64 | 整型     | 8    | 0     |                                                  |
| float32       | 浮点型   | 4    | 0.0   | 小数位精确到7位                                  |
| float64       | 浮点型   | 8    | 0.0   | 小数位精确到15位                                 |
| complex64     | 复数类型 | 8    |       |                                                  |
| complex128    | 复数类型 | 16   |       |                                                  |
| uintptr       | 整型     | 4/8  |       | ⾜以存储指针的 uint32或 uint64整数               |
| string        | 字符串   |      | "“    | utf-8字符串                                      |

### Byte&Rune

一个byte变量大小为1字节，用于表示ACSCII码字符，相当于`unit8`类型
rune变量大小为4字节，用于表示采用 UTF-8 编码的Unicode字符，相当于`uint32`类型

## Slice&Array

数组为具有**固定长度**的基本数据结构，长度大于元素个数的时候用0补位，*不能用make初始化，不能动态设定长度*
array初始化：
```Go
var arr = [constant]int{a,b,c}
var arr = [...]int{a,b,c}//数组长度由初始化元素个数决定
```

slice本质上为对数组的**引用**，对slice 的改变等于直接改变数组
slice作为参数传递给函数时进行浅拷贝，函数对slice的操作实际为对底层数组的操作
```Go
type slice struct{
array unsafe.Pointer
length int
capacity int
}
```

slice初始化：
```Go
// len(slice1)=high-low cap(slice1)=len(array)-low
slice1 := array[low:high:max]// max 不能超过原数组长度
slice2 := make([]type,length,capacity)
slice3 := []int{a,b,c}
```

```Go
var s []string             // nil slice
var s = make([]string, 0)  // empty slice
```

当使用`append`对slice进行追加时，如果超出原数组大小，go会拷贝创建一个更大的新数组，令slice变为新数组的引用
```Go
// "..."将slice2打散为元素加入slice1
slice1 = append(slice1,slice2...)
```

3种特殊slice类型：
- 0-slice：slice元素未赋值，都是类型0值
- empty-slice：`len(slice)=empty(slice)=0` 底层数组指针为特殊指针`zerobase`
- nil-slice：未初始化slice，底层数组指针

### 扩容

V1.18以后：
```Go
// src/runtime/slice.go
func growslice(et *_type, old slice, cap int) slice {
   newcap := old.cap
   doublecap := newcap + newcap
   if cap > doublecap {
      // 新容量大于两倍旧容量则直接扩容至新容量
      newcap = cap
   } else {
      const threshold = 256
      if old.cap < threshold {
         newcap = doublecap
      } else {
         // Check 0 < newcap to detect overflow
         // and prevent an infinite loop.
         for 0 < newcap && newcap < cap {
            // Transition from growing 2x for small slices
            // to growing 1.25x for large slices. This formula
            // gives a smooth-ish transition between the two.
            newcap += (newcap + 3*threshold) / 4
         }
         // Set newcap to the requested cap when
         // the newcap calculation overflowed.
         if newcap <= 0 {
            newcap = cap
         }
      }
   }
  
  
  //.....
  // Specialize for common values of et.size.
  // For 1 we don't need any division/multiplication.
  // For goarch.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
  // For powers of 2, use a variable shift.
    switch {
    case et.size == 1:
        //...
    case et.size == goarch.PtrSize:
       //...
    default:
       lenmem = uintptr(old.len) * et.size
       newlenmem = uintptr(cap) * et.size
       capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
       capmem = roundupsize(capmem)
       newcap = int(capmem / et.size)
    }
   }
```

## String

>[Strings, bytes, runes and characters in Go](https://go.dev/blog/strings)

String 类型对应一个 struct
该 struct 包含两个元素，第一个是指向该 string 的字节数组的指针，第二个是 string 的长度，每个元素占8个字节
```Go
type StringHeader struct {
    Data uintptr // 指向字节数组的指针
    Len  int     // string的长度
}
```

- string 初始值为空字符串 `""`，但 `"" != nil 
- String 指向字符串字面量，不允许修改，但可以通过下标遍历 
- String 的赋值操作仅为结构体复制，不会涉及底层字节数组的复制 
- String 是8bit 字节的集合，通常是 UTF-8编码的文本
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

反引号声明的string中的特殊字符不需转义，所见即所得

## Map

> [Go Map底层实现原理](https://zhuanlan.zhihu.com/p/495998623)

<?xml version="1.0" encoding="UTF-8"?>
<!-- Do not edit this file with editors other than diagrams.net -->
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1" width="981px" height="581px" viewBox="-0.5 -0.5 981 581" content="&lt;mxfile host=&quot;app.diagrams.net&quot; modified=&quot;2023-03-10T04:03:08.237Z&quot; agent=&quot;5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36 Edg/111.0.0.0&quot; version=&quot;21.0.4&quot; etag=&quot;zFuT0-pcnISe55yDOliL&quot; type=&quot;onedrive&quot;&gt;&lt;diagram id=&quot;uasperhxpneqPqqxGoWr&quot; name=&quot;第 1 页&quot;&gt;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&lt;/diagram&gt;&lt;/mxfile&gt;" resource="https://app.diagrams.net/#Wed16f148a30900a%2FED16F148A30900A!61129"><defs/><g><rect x="0" y="0" width="980" height="580" fill="#ffffff" stroke="#000000" pointer-events="all"/><rect x="30" y="155" width="150" height="280" fill="#ffffff" stroke="#000000" stroke-dasharray="3 3" pointer-events="all"/><rect x="75" y="165" width="60" height="30" fill="none" stroke="none" pointer-events="all"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 180px; margin-left: 76px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 20px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">hmap</div></div></div></foreignObject><text x="105" y="186" fill="#000000" font-family="Helvetica" font-size="20px" text-anchor="middle">hmap</text></switch></g><rect x="50" y="205" width="110" height="200" fill="#ffe6cc" stroke="none" pointer-events="none"/><path d="M 50 205 L 160 205 L 160 405 L 50 405 L 50 205" fill="none" stroke="#d79b00" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><path d="M 50 245 L 160 245" fill="none" stroke="#d79b00" stroke-miterlimit="10" pointer-events="none"/><path d="M 50 285 L 160 285" fill="none" stroke="#d79b00" stroke-miterlimit="10" pointer-events="none"/><path d="M 50 325 L 160 325" fill="none" stroke="#d79b00" stroke-miterlimit="10" pointer-events="none"/><path d="M 50 365 L 160 365" fill="none" stroke="#d79b00" stroke-miterlimit="10" pointer-events="none"/><path d="M 50 205 M 160 205 M 160 245 M 50 245" fill="none" stroke="#d79b00" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 225px; margin-left: 51px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 36px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">B</div></div></div></foreignObject><text x="105" y="230" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">B</text></switch></g><path d="M 50 245 M 160 245 M 160 285 M 50 285" fill="none" stroke="#d79b00" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 265px; margin-left: 51px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 36px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">buckets</div></div></div></foreignObject><text x="105" y="270" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">buckets</text></switch></g><path d="M 50 285 M 160 285 M 160 325 M 50 325" fill="none" stroke="#d79b00" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 305px; margin-left: 51px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 36px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">oldbuckets</div></div></div></foreignObject><text x="105" y="310" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">oldbuckets</text></switch></g><path d="M 50 325 M 160 325 M 160 365 M 50 365" fill="none" stroke="#d79b00" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 345px; margin-left: 51px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 36px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">extra</div></div></div></foreignObject><text x="105" y="350" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">extra</text></switch></g><path d="M 50 365 M 160 365 M 160 405 M 50 405" fill="none" stroke="#d79b00" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 385px; margin-left: 51px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 36px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">...</div></div></div></foreignObject><text x="105" y="390" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">...</text></switch></g><rect x="220" y="50" width="130" height="220" fill="#ffffff" stroke="#000000" stroke-dasharray="3 3" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 75px; margin-left: 256px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 20px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">buckets</div></div></div></foreignObject><text x="285" y="81" fill="#000000" font-family="Helvetica" font-size="20px" text-anchor="middle">buckets</text></switch></g><rect x="230" y="100" width="110" height="140" fill="#dae8fc" stroke="none" pointer-events="none"/><path d="M 230 100 L 340 100 L 340 240 L 230 240 L 230 100" fill="none" stroke="#6c8ebf" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><path d="M 230 147 L 340 147" fill="none" stroke="#6c8ebf" stroke-miterlimit="10" pointer-events="none"/><path d="M 230 193 L 340 193" fill="none" stroke="#6c8ebf" stroke-miterlimit="10" pointer-events="none"/><path d="M 230 100 M 340 100 M 340 147 M 230 147" fill="none" stroke="#6c8ebf" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 124px; margin-left: 231px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 43px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">[0]bmap</div></div></div></foreignObject><text x="285" y="128" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">[0]bmap</text></switch></g><path d="M 230 147 M 340 147 M 340 193 M 230 193" fill="none" stroke="#6c8ebf" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 170px; margin-left: 231px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 42px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">...</div></div></div></foreignObject><text x="285" y="175" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">...</text></switch></g><path d="M 230 193 M 340 193 M 340 240 M 230 240" fill="none" stroke="#6c8ebf" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 217px; margin-left: 231px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 43px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">[7]bmap</div></div></div></foreignObject><text x="285" y="221" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">[7]bmap</text></switch></g><rect x="220" y="320" width="130" height="220" fill="#ffffff" stroke="#000000" stroke-dasharray="3 3" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 340px; margin-left: 256px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 20px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">extra</div></div></div></foreignObject><text x="285" y="346" fill="#000000" font-family="Helvetica" font-size="20px" text-anchor="middle">extra</text></switch></g><rect x="235" y="370" width="100" height="140" fill="#dae8fc" stroke="none" pointer-events="none"/><path d="M 235 370 L 335 370 L 335 510 L 235 510 L 235 370" fill="none" stroke="#6c8ebf" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><path d="M 235 417 L 335 417" fill="none" stroke="#6c8ebf" stroke-miterlimit="10" pointer-events="none"/><path d="M 235 463 L 335 463" fill="none" stroke="#6c8ebf" stroke-miterlimit="10" pointer-events="none"/><path d="M 235 370 M 335 370 M 335 417 M 235 417" fill="none" stroke="#6c8ebf" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 98px; height: 1px; padding-top: 394px; margin-left: 236px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 43px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">overflow</div></div></div></foreignObject><text x="285" y="398" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">overflow</text></switch></g><path d="M 235 417 M 335 417 M 335 463 M 235 463" fill="none" stroke="#6c8ebf" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 98px; height: 1px; padding-top: 440px; margin-left: 236px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 42px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">oldOverflow</div></div></div></foreignObject><text x="285" y="445" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">oldOverflow</text></switch></g><path d="M 235 463 M 335 463 M 335 510 M 235 510" fill="none" stroke="#6c8ebf" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 98px; height: 1px; padding-top: 487px; margin-left: 236px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 43px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">nextOverflow</div></div></div></foreignObject><text x="285" y="491" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">nextOverflow</text></switch></g><rect x="390" y="85" width="266" height="410" fill="#ffffff" stroke="#000000" stroke-dasharray="3 3" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 110px; margin-left: 494px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 20px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">bmap</div></div></div></foreignObject><text x="523" y="116" fill="#000000" font-family="Helvetica" font-size="20px" text-anchor="middle">bmap</text></switch></g><rect x="410" y="135" width="56" height="260" fill="#60a917" stroke="none" pointer-events="none"/><path d="M 410 135 L 466 135 L 466 395 L 410 395 L 410 135" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><path d="M 410 169 L 466 169" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 410 201 L 466 201" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 410 233 L 466 233" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 410 265 L 466 265" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 410 297 L 466 297" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 410 329 L 466 329" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 410 361 L 466 361" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 410 135 M 466 135 M 466 169 M 410 169" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 152px; margin-left: 411px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;"><font style="font-size: 14px;">tophash</font></div></div></div></foreignObject><text x="438" y="157" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">tophash</text></switch></g><path d="M 410 169 M 466 169 M 466 201 M 410 201" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 185px; margin-left: 411px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">10111011</div></div></div></foreignObject><text x="438" y="189" fill="#000000" font-family="Helvetica" font-size="12px" text-anchor="middle">10111011</text></switch></g><path d="M 410 201 M 466 201 M 466 233 M 410 233" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 217px; margin-left: 411px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">10101011</div></div></div></foreignObject><text x="438" y="221" fill="#000000" font-family="Helvetica" font-size="12px" text-anchor="middle">10101011</text></switch></g><path d="M 410 233 M 466 233 M 466 265 M 410 265" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 249px; margin-left: 411px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">11111110</div></div></div></foreignObject><text x="438" y="253" fill="#000000" font-family="Helvetica" font-size="12px" text-anchor="middle">11111110</text></switch></g><path d="M 410 265 M 466 265 M 466 297 M 410 297" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 281px; margin-left: 411px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">11011010</div></div></div></foreignObject><text x="438" y="285" fill="#000000" font-family="Helvetica" font-size="12px" text-anchor="middle">11011010</text></switch></g><path d="M 410 297 M 466 297 M 466 329 M 410 329" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 313px; margin-left: 411px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">10110101</div></div></div></foreignObject><text x="438" y="317" fill="#000000" font-family="Helvetica" font-size="12px" text-anchor="middle">10110101</text></switch></g><path d="M 410 329 M 466 329 M 466 361 M 410 361" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 345px; margin-left: 411px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">11000100</div></div></div></foreignObject><text x="438" y="349" fill="#000000" font-family="Helvetica" font-size="12px" text-anchor="middle">11000100</text></switch></g><path d="M 410 361 M 466 361 M 466 395 M 410 395" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 378px; margin-left: 411px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">10100011</div></div></div></foreignObject><text x="438" y="382" fill="#000000" font-family="Helvetica" font-size="12px" text-anchor="middle">10100011</text></switch></g><path d="M 160 345 L 190 345 L 190 430 L 208.03 430" fill="none" stroke="#ff0505" stroke-width="4" stroke-miterlimit="10" pointer-events="none"/><path d="M 215.53 430 L 205.53 435 L 208.03 430 L 205.53 425 Z" fill="#ff0505" stroke="#ff0505" stroke-width="4" stroke-miterlimit="10" pointer-events="none"/><path d="M 160 265 L 190 265 L 190 160 L 208.03 160" fill="none" stroke="#ff0505" stroke-width="4" stroke-miterlimit="10" pointer-events="none"/><path d="M 215.53 160 L 205.53 165 L 208.03 160 L 205.53 155 Z" fill="#ff0505" stroke="#ff0505" stroke-width="4" stroke-miterlimit="10" pointer-events="none"/><path d="M 583 450 L 634.5 450 L 634.5 290 L 674.03 290" fill="none" stroke="#ff0505" stroke-width="4" stroke-miterlimit="10" pointer-events="none"/><path d="M 681.53 290 L 671.53 293.33 L 674.03 290 L 671.53 286.67 Z" fill="#ff0505" stroke="#ff0505" stroke-width="4" stroke-miterlimit="10" pointer-events="none"/><rect x="463" y="430" width="120" height="40" fill="#60a917" stroke="#2d7600" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 118px; height: 1px; padding-top: 450px; margin-left: 464px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 14px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;"><font>overflow</font></div></div></div></foreignObject><text x="523" y="454" fill="#000000" font-family="Helvetica" font-size="14px" text-anchor="middle">overflow</text></switch></g><rect x="486" y="135" width="56" height="260" fill="#60a917" stroke="none" pointer-events="none"/><path d="M 486 135 L 542 135 L 542 395 L 486 395 L 486 135" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><path d="M 486 169 L 542 169" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 486 202 L 542 202" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 486 234 L 542 234" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 486 266 L 542 266" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 486 296 L 542 296" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 486 328 L 542 328" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 486 361 L 542 361" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 486 135 M 542 135 M 542 169 M 486 169" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 152px; margin-left: 487px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;"><font style="font-size: 14px;">keys</font></div></div></div></foreignObject><text x="514" y="157" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">keys</text></switch></g><path d="M 486 169 M 542 169 M 542 202 M 486 202" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 186px; margin-left: 487px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 29px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">0</div></div></div></foreignObject><text x="514" y="190" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">0</text></switch></g><path d="M 486 202 M 542 202 M 542 234 M 486 234" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 218px; margin-left: 487px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">1</div></div></div></foreignObject><text x="514" y="223" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">1</text></switch></g><path d="M 486 234 M 542 234 M 542 266 M 486 266" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 250px; margin-left: 487px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">2</div></div></div></foreignObject><text x="514" y="255" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">2</text></switch></g><path d="M 486 266 M 542 266 M 542 296 M 486 296" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 281px; margin-left: 487px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 26px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">3</div></div></div></foreignObject><text x="514" y="286" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">3</text></switch></g><path d="M 486 296 M 542 296 M 542 328 M 486 328" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 312px; margin-left: 487px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">4</div></div></div></foreignObject><text x="514" y="317" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">4</text></switch></g><path d="M 486 328 M 542 328 M 542 361 M 486 361" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 345px; margin-left: 487px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 29px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">5</div></div></div></foreignObject><text x="514" y="349" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">5</text></switch></g><path d="M 486 361 M 542 361 M 542 395 M 486 395" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 378px; margin-left: 487px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">6</div></div></div></foreignObject><text x="514" y="383" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">6</text></switch></g><rect x="566" y="135" width="56" height="260" fill="#60a917" stroke="none" pointer-events="none"/><path d="M 566 135 L 622 135 L 622 395 L 566 395 L 566 135" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><path d="M 566 169 L 622 169" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 566 202 L 622 202" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 566 234 L 622 234" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 566 266 L 622 266" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 566 296 L 622 296" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 566 328 L 622 328" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 566 361 L 622 361" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 566 135 M 622 135 M 622 169 M 566 169" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 152px; margin-left: 567px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;"><font style="font-size: 14px;">values</font></div></div></div></foreignObject><text x="594" y="157" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">values</text></switch></g><path d="M 566 169 M 622 169 M 622 202 M 566 202" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 186px; margin-left: 567px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 29px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">0</div></div></div></foreignObject><text x="594" y="190" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">0</text></switch></g><path d="M 566 202 M 622 202 M 622 234 M 566 234" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 218px; margin-left: 567px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">1</div></div></div></foreignObject><text x="594" y="223" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">1</text></switch></g><path d="M 566 234 M 622 234 M 622 266 M 566 266" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 250px; margin-left: 567px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">2</div></div></div></foreignObject><text x="594" y="255" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">2</text></switch></g><path d="M 566 266 M 622 266 M 622 296 M 566 296" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 281px; margin-left: 567px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 26px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">3</div></div></div></foreignObject><text x="594" y="286" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">3</text></switch></g><path d="M 566 296 M 622 296 M 622 328 M 566 328" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 312px; margin-left: 567px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">4</div></div></div></foreignObject><text x="594" y="317" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">4</text></switch></g><path d="M 566 328 M 622 328 M 622 361 M 566 361" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 345px; margin-left: 567px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 29px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">5</div></div></div></foreignObject><text x="594" y="349" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">5</text></switch></g><path d="M 566 361 M 622 361 M 622 395 M 566 395" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 378px; margin-left: 567px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">6</div></div></div></foreignObject><text x="594" y="383" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">6</text></switch></g><path d="M 472.37 281 L 479.63 281" fill="none" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 467.12 281 L 474.12 278.67 L 472.37 281 L 474.12 283.33 Z" fill="#ff0505" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 484.88 281 L 477.88 283.33 L 479.63 281 L 477.88 278.67 Z" fill="#ff0505" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 548.37 281 L 559.63 281" fill="none" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 543.12 281 L 550.12 278.67 L 548.37 281 L 550.12 283.33 Z" fill="#ff0505" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 564.88 281 L 557.88 283.33 L 559.63 281 L 557.88 278.67 Z" fill="#ff0505" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><rect x="686" y="85" width="266" height="410" fill="#ffffff" stroke="#000000" stroke-dasharray="3 3" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 110px; margin-left: 790px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 20px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">bmap</div></div></div></foreignObject><text x="819" y="116" fill="#000000" font-family="Helvetica" font-size="20px" text-anchor="middle">bmap</text></switch></g><rect x="706" y="135" width="56" height="260" fill="#60a917" stroke="none" pointer-events="none"/><path d="M 706 135 L 762 135 L 762 395 L 706 395 L 706 135" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><path d="M 706 169 L 762 169" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 706 201 L 762 201" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 706 233 L 762 233" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 706 265 L 762 265" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 706 297 L 762 297" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 706 329 L 762 329" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 706 361 L 762 361" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 706 135 M 762 135 M 762 169 M 706 169" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 152px; margin-left: 707px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;"><font style="font-size: 14px;">tophash</font></div></div></div></foreignObject><text x="734" y="157" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">tophash</text></switch></g><path d="M 706 169 M 762 169 M 762 201 M 706 201" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 185px; margin-left: 707px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">...</div></div></div></foreignObject><text x="734" y="190" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">...</text></switch></g><path d="M 706 201 M 762 201 M 762 233 M 706 233" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 217px; margin-left: 707px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">...</div></div></div></foreignObject><text x="734" y="222" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">...</text></switch></g><path d="M 706 233 M 762 233 M 762 265 M 706 265" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 249px; margin-left: 707px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">...</div></div></div></foreignObject><text x="734" y="254" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">...</text></switch></g><path d="M 706 265 M 762 265 M 762 297 M 706 297" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 281px; margin-left: 707px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">...</div></div></div></foreignObject><text x="734" y="286" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">...</text></switch></g><path d="M 706 297 M 762 297 M 762 329 M 706 329" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 313px; margin-left: 707px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">...</div></div></div></foreignObject><text x="734" y="318" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">...</text></switch></g><path d="M 706 329 M 762 329 M 762 361 M 706 361" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 345px; margin-left: 707px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">...</div></div></div></foreignObject><text x="734" y="350" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">...</text></switch></g><path d="M 706 361 M 762 361 M 762 395 M 706 395" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 378px; margin-left: 707px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">...</div></div></div></foreignObject><text x="734" y="383" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">...</text></switch></g><rect x="759" y="430" width="120" height="40" fill="#60a917" stroke="#2d7600" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 118px; height: 1px; padding-top: 450px; margin-left: 760px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 14px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">overflow</div></div></div></foreignObject><text x="819" y="454" fill="#000000" font-family="Helvetica" font-size="14px" text-anchor="middle">overflow</text></switch></g><rect x="782" y="135" width="56" height="260" fill="#60a917" stroke="none" pointer-events="none"/><path d="M 782 135 L 838 135 L 838 395 L 782 395 L 782 135" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><path d="M 782 169 L 838 169" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 782 202 L 838 202" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 782 234 L 838 234" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 782 266 L 838 266" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 782 296 L 838 296" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 782 328 L 838 328" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 782 361 L 838 361" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 782 135 M 838 135 M 838 169 M 782 169" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 152px; margin-left: 783px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;"><font style="font-size: 14px;">keys</font></div></div></div></foreignObject><text x="810" y="157" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">keys</text></switch></g><path d="M 782 169 M 838 169 M 838 202 M 782 202" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 186px; margin-left: 783px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 29px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">0</div></div></div></foreignObject><text x="810" y="190" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">0</text></switch></g><path d="M 782 202 M 838 202 M 838 234 M 782 234" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 218px; margin-left: 783px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">1</div></div></div></foreignObject><text x="810" y="223" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">1</text></switch></g><path d="M 782 234 M 838 234 M 838 266 M 782 266" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 250px; margin-left: 783px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">2</div></div></div></foreignObject><text x="810" y="255" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">2</text></switch></g><path d="M 782 266 M 838 266 M 838 296 M 782 296" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 281px; margin-left: 783px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 26px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">3</div></div></div></foreignObject><text x="810" y="286" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">3</text></switch></g><path d="M 782 296 M 838 296 M 838 328 M 782 328" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 312px; margin-left: 783px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">4</div></div></div></foreignObject><text x="810" y="317" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">4</text></switch></g><path d="M 782 328 M 838 328 M 838 361 M 782 361" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 345px; margin-left: 783px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 29px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">5</div></div></div></foreignObject><text x="810" y="349" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">5</text></switch></g><path d="M 782 361 M 838 361 M 838 395 M 782 395" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 378px; margin-left: 783px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">6</div></div></div></foreignObject><text x="810" y="383" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">6</text></switch></g><rect x="862" y="135" width="56" height="260" fill="#60a917" stroke="none" pointer-events="none"/><path d="M 862 135 L 918 135 L 918 395 L 862 395 L 862 135" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><path d="M 862 169 L 918 169" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 862 202 L 918 202" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 862 234 L 918 234" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 862 266 L 918 266" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 862 296 L 918 296" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 862 328 L 918 328" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 862 361 L 918 361" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><path d="M 862 135 M 918 135 M 918 169 M 862 169" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 152px; margin-left: 863px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;"><font style="font-size: 14px;">values</font></div></div></div></foreignObject><text x="890" y="157" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">values</text></switch></g><path d="M 862 169 M 918 169 M 918 202 M 862 202" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 186px; margin-left: 863px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 29px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">0</div></div></div></foreignObject><text x="890" y="190" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">0</text></switch></g><path d="M 862 202 M 918 202 M 918 234 M 862 234" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 218px; margin-left: 863px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">1</div></div></div></foreignObject><text x="890" y="223" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">1</text></switch></g><path d="M 862 234 M 918 234 M 918 266 M 862 266" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 250px; margin-left: 863px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">2</div></div></div></foreignObject><text x="890" y="255" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">2</text></switch></g><path d="M 862 266 M 918 266 M 918 296 M 862 296" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 281px; margin-left: 863px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 26px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">3</div></div></div></foreignObject><text x="890" y="286" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">3</text></switch></g><path d="M 862 296 M 918 296 M 918 328 M 862 328" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 312px; margin-left: 863px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 28px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">4</div></div></div></foreignObject><text x="890" y="317" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">4</text></switch></g><path d="M 862 328 M 918 328 M 918 361 M 862 361" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 345px; margin-left: 863px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 29px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">5</div></div></div></foreignObject><text x="890" y="349" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">5</text></switch></g><path d="M 862 361 M 918 361 M 918 395 M 862 395" fill="none" stroke="#2d7600" stroke-linecap="square" stroke-miterlimit="10" pointer-events="none"/><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 54px; height: 1px; padding-top: 378px; margin-left: 863px;"><div data-drawio-colors="color: #000000; " style="box-sizing: border-box; font-size: 0px; text-align: center; max-height: 30px; overflow: hidden;"><div style="display: inline-block; font-size: 16px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: none; white-space: normal; overflow-wrap: normal;">6</div></div></div></foreignObject><text x="890" y="383" fill="#000000" font-family="Helvetica" font-size="16px" text-anchor="middle">6</text></switch></g><path d="M 768.37 281 L 775.63 281" fill="none" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 763.12 281 L 770.12 278.67 L 768.37 281 L 770.12 283.33 Z" fill="#ff0505" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 780.88 281 L 773.88 283.33 L 775.63 281 L 773.88 278.67 Z" fill="#ff0505" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 844.37 281 L 855.63 281" fill="none" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 839.12 281 L 846.12 278.67 L 844.37 281 L 846.12 283.33 Z" fill="#ff0505" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 860.88 281 L 853.88 283.33 L 855.63 281 L 853.88 278.67 Z" fill="#ff0505" stroke="#ff0505" stroke-miterlimit="10" pointer-events="none"/><path d="M 340 123.5 L 365 123.5 L 365 290 L 378.03 290" fill="none" stroke="#ff0505" stroke-width="4" stroke-miterlimit="10" pointer-events="none"/><path d="M 385.53 290 L 375.53 293.33 L 378.03 290 L 375.53 286.67 Z" fill="#ff0505" stroke="#ff0505" stroke-width="4" stroke-miterlimit="10" pointer-events="none"/></g><switch><g requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"/><a transform="translate(0,-5)" xlink:href="https://www.diagrams.net/doc/faq/svg-export-text-problems" target="_blank"><text text-anchor="middle" font-size="10px" x="50%" y="100%">Text is not SVG - cannot display</text></a></switch></svg>

**map 为引用类型**，用 make 函数创建初始化 map 时会在堆上分配一个 `runtime.hmap` 类型的数据结构，并返回指向这块内存区域的指针
```go
type hmap struct {
	count     int        // 元素的个数
	B         uint8      // bucket数量的对数 
	flags     uint8      //
	noverflow uint16     // 溢出桶的大概数量
	hash0     uint32     // hash种子，为哈希函数的结果引入随机性

	
	buckets    unsafe.Pointer // 2^B个桶对应的指针数组
	oldbuckets unsafe.Pointer // 扩容时记录旧buckets数组的指针
	nevacuate  uintptr        // 疏散进度计数器（小于此的桶已被疏散)

	extra *mapextra  // 用于保存溢出桶的地址
}

type mapextra struct {
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	nextOverflow *bmap   // 保存一个指向空闲溢出桶的指针
}

type bmap struct {
     tophash [bucketCnt]uint8
 }

// 在编译期间会产生新的结构体
type bmap struct {
    topbits  [8]uint8      //存储哈希值的高8位
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr       // 溢出bucket的地址
}
```

Map 中单个哈希桶 `bmap` 最多存储8个元素，溢出后新建溢出桶存储，使用 `overflow` 指向溢出桶位置

语法
```Go
// 声明&初始化
// 只是声明一个map，没有初始化，m1=nil，此时赋值引发panic
var m1 map[type]type
// 初始化为空map
m1 := map[type]type{}
// 同上，初始化为空map
m1 := make(map[int]string, 0)

//操作
// 赋值，自动为map分配空间，但此时map需已初始化，不能为nil
m1[a] = b   
// 取值，键值不存在时ok=false
value, ok = m1[a]
// 删除指定key
delete(m1, a)
```

> [!attention]
map并非并发安全，并发读写map会导致panic且该panic无法被recover捕获

### 查找

1. key经过哈希计算后得到哈希值，共64个bit位
2. 根据最后 B(*bucket 数量的对数*)位确定桶编号
3. 使用哈希值高8位比对桶内 `tophash` 数组查找值，有匹配则进一步比对完整 key 值，相同返回对应 value
4. 若无匹配且 `overflow` 指针不为空，去溢出桶内继续匹配

### 扩容

步骤：
1. 将当前桶数组由 `buckets` 指针转移到 `oldbuckets` 指针
2. 创建新桶数组
3. 每次对 map 进行写操作时触发从 oldbucket 中迁移部分数据到 bucket 的增量操作
4. 在扩容没有完全迁移完成之前，每次查找数据时先遍历 oldbuckets 再遍历 buckets

*等量扩容*：创建等量新桶，桶内元素重排，填满前置空位，删除空桶
触发时机：溢出的桶太多，溢出桶的数量大于等于 2B 时触发等量扩容
等量扩容不会使桶内元素顺序变化

*成倍扩容*：创建两倍数量新桶，B 增加，元素可能发生桶迁移
触发时机： map 写操作时装载因子(元素数量/bucket 数量) > 6.5
成倍扩容会使桶内元素顺序变化

因为 Go 语言哈希的扩容不是一个原子的过程，所以**两种扩容都需要判断当前map是否已经处于扩容状态**，避免二次扩容造成混乱

### 遍历

**Map 每次遍历输出元素的顺序不一致**
Go 通过使 for range 遍历 Map 的索引起点随机强制要求程序不依赖 map 的顺序
本质是因为 key 写入 map 时是随机的，扩容时也会发生无序移动，而 map 也没有另外对键值对的顺序进行维护

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

通过在 struct 内加入匿名字段可以以隐式继承该变量的方法及属性
```Go
type A

func (a *A) methodA()

type B struct {
	A
}

B.methodA()
```

### Empty struct

**空结构体不占据内存空间**，因此可以用于占位符节省资源

- 可以通过 `map[int]struct{}` 方式实现集合
- 使用空结构体作为 channel 占位符
- 空结构体构建只包含方法不占用空间的结构体

> [Go 空结构体 struct{} 的使用 | Go 语言高性能编程 | 极客兔兔 (geektutu.com)](https://geektutu.com/post/hpg-empty-struct.html)

## Channel

> [Go 语言 Channel 实现原理精要 | Go 语言设计与实现 (draveness.me)](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)

Channel 用于 goroutine 之间通信，充当一个先进先出的队列
- 先从 Channel 读取数据的 Goroutine 会先接收到数据
- 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权利

### 特性

- 对一个 nil channel 进行读写会引发协程的永久阻塞
- 关闭一个未初始化或已经关闭的 channel 会产生 panic
- 向一个已关闭的 channel 发送消息会产生 panic
- 从一个已关闭的 channel 读取消息不会发生 panic，读取完缓存数据后会一直读取到零值
- Channel 可以读端和写端都可有多个 goroutine 操作，在一端关闭 channel 的时候，该 channel 读端的所有 goroutine 都会收到 channel 已关闭的消息
- Channel 是并发安全的，多个 goroutine 同时读取 channel 中的数据，不会产生并发安全问题
- Channel 可以相互比较，两个不是 nil 的 channel 比较实际上比较的是他们是否引用同一个对象

### 数据结构

Channel 是一个**引用类型**，用 make 函数创建初始化 channel 时会在堆上分配一个 `runtime.hchan` 类型的数据结构，并返回指向这块内存区域的指针
```Go
type hchan struct {
   qcount   uint           // 循环队列中的数据总数
   dataqsiz uint           // 循环队列大小
   buf      unsafe.Pointer // 指向循环队列的指针
   elemsize uint16         // 循环队列中的每个元素的大小
   closed   uint32         // 标记位，标记channel是否关闭
   elemtype *_type         // 循环队列中的元素类型
   sendx    uint           // 当前可以存储发送元素的位置索引值
   recvx    uint           // 当前可以释放接收元素的位置索引值
   recvq    waitq          // 等待从channel接收消息的sudog队列
   sendq    waitq          // 等待向channel写入消息的sudog队列
   lock mutex              // 互斥锁，对channel的数据读写操作加锁，保证并发安全
}

type waitq struct {
    first *sudog           // sudog队列的队头指针
    last  *sudog           // sudog队列的队尾指针
}

type sudog struct {
   g *g                    // 绑定的goroutine
   next *sudog             // 指向sudog链表中的下一个节点
   prev *sudog             // 指向sudog链表中的下一个节点
   elem unsafe.Pointer     // 数据对象
   acquiretime int64     
   releasetime int64
   ticket      uint32
   isSelect bool
   success bool
   parent   *sudog         // semaRoot binary tree
   waitlink *sudog         // g.waiting list or semaRoot
   waittail *sudog         // semaRoot
   c        *hchan         // channel
}

```

`hchan` 使用 buf 指向实际存储缓存的元素数组，同时使用 `recvq` 以及 `sendq` 存储由于缓冲区空间不足被阻塞的协程队列
`waitq` 是一个 sudog 队列的双向链表
`sudog` 是对 goroutine 的一个封装

无缓冲 channel 在读写的时候是阻塞的
有缓冲 channel 在缓冲队列满了之后写入阻塞，队列空时读取阻塞

读取：
```go
value := <-ch      //仅接收数据
value,ok := <-ch   //接收数据以及代表操作成功与否的bool值
```

Channel 结束使用后需要 close，避免程序一直在等待以及资源的浪费
可以从被 close 的 channel 中接收数据，**缓存数据接收完后将继续接收到零值**，此时 `ok` 为 false

### 初始化

与 map 一样，channel 需要初始化：
```Go
ch := make(chan string)           // 无缓冲channel
ch := make(chan string, capacity) // 有缓冲channel
```

```go
// https://github.com/golang/go/blob/8edcdddb23c6d3f786b465c43b49e8d9a0015082/src/runtime/chan.go#L72

func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0: // 不存在缓冲区，只为hchan分配一段内存空间
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0: // 元素类型不含指针, 只进行一次内存分配
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default: // 默认情况下会单独为hchan和缓冲区分配内存
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)  // 循环数组长度
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c  // 返回 hchan 指针
}
```

### 并发安全

当使用 `send (ch <- xx)` 或者 `recv ( <-ch)` 的时候，首先通过 `hchan.lock` 锁住 ` hchan ` 这个结构体，然后再读写

## Interface

接口是一组方法的声明的集合，依托于自定义类型对方法的具体实现而存在
实现了接口中所有方法的类型就可以为该接口的实例赋值，该接口的实例仅可调用接口中的方法

```Go
type interfaceName interface {
   methodName1([parameter_list]) [return_type_list]
   methodName2([parameter_list]) [return_type_list]
   methodName3([parameter_list]) [return_type_list]
   ...
}
```
interface同样可作为匿名字段嵌入其他struct，超集接口对象可转换为子集接口对象，反之出错

没有任何方法声明的接口为空接口`interface{}`，因此所有类型都实现了空接口，空接口是任意对象的子集，进而可以使用任意类型的数值赋值空接口

### 断言

另一方面，使用接口为其他类型变量赋值时需使用**断言**
断言通过以下表达式判断`element`变量是否为类型T，是则返回类型为T的对象值
```Go
value, ok := element.(T)
```
`ok`为bool值，如果直接使用`value = x.(T)`，断言失败将导致程序报`panic`

## 类型比较

> [Golang 之 struct能不能比较 - 掘金 (juejin.cn)](https://juejin.cn/post/6881912621616857102)

Go 语言数据类型比较：
- 可直接比较：_Integer_，_Floating-point_，_String_，_Boolean_，_Complex(复数型)_，_Pointer_，_Channel_，_Interface_，_Array_
- 不可直接比较：_Slice_，_Map_，_Function_

对于 struct
- 同类型 struct：当 struct 不包含不可直接比较成员变量时可直接比较，否则不可直接比较
- 不同类型 struct：可通过强制转换进行比较，同样需不包含不可直接比较的成员变量

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

**struct 必须是可比较的，才能作为 map 的 key**
