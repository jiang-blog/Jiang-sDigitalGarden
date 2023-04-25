---
{"dg-publish":true,"permalink":"/Code/6.Go/Go.0b.Go语法/","title":"Go 语法","noteIcon":""}
---


# Go 语法

## 函数

### For

`for index,value = range(array)`，类似于
```Go
len_temp := len(array)
array_temp := array
var index int
value := array[0]
for index_temp = 0; index_temp < len_temp; index_temp++ {
     value_temp = array_temp[index_temp]
     index = index_temp
     value = value_temp
     /*
      * original body
      */              
 }  
```
因此具有如下特点：
1. 在循环中value的地址始终不变
2. 循环开始后index和value的值确定，不再受原始array的更改影响
3. for range遍历的内容是对原内容的一个拷贝，所以不能用来修改原切片中内容

### Make&new

make 函数只用于 map，slice 和 channel初始化，并且不返回指针，因为这三种类型本身即为引用类型
new 函数只接受一个类型参数，并且返回一个指向该类型内存地址的指针，且将其中类型初始化为0值，指针初始化为nil

### 匿名函数&闭包

>[!Cite] 参考
> [Golang：“闭包（closure）”到底包了什么？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/92634505)

闭包通常通过调用一个**外部函数**返回其**内部匿名函数**来实现

在闭包**第一次实际运行**时，其保存相关的引用环境(闭包捕获的变量和常量都是引用传递，不是值传递)，直到闭包的生命周期结束

**延迟绑定**：闭包内变量的初始值为闭包**第一次实际运行**（而非定义）时变量的最新值，同时闭包内的变量不随原始变量本身的生命周期结束而消失
- 当闭包内变量为指针时，也同时保证了指针指向的值不消失

### Method 方法

Method 是绑定在某种类型的变量(结构体或基础类型的别名)上的特殊的函数

```Go
func (recv recvType) methodName(paramrList) (returnValList) { ... }
```

`recvType` 既可以为值也可以为指针
Method 既可以通过变量指针调用也可以通过变量调用，调用方法相同，都为 `recv.methodName(paramrList)` 
但仅当 `recvType` 为指针时，method 才能改变 `recv` 的值
当 `recvType` 不是指针时，method 针对接收者值的一个副本进行操作

**在进行多态调用的时候，必须满足如下规则**
| `recvType` 的类型 | 参数的类型 |
| ---------------- | ---------- |
| 值               | 值或指针   |
| 指针             | 指针           |

### Defer

> [【GoLang】defer的坑与应用 - 第一节_Gnight_jmup的博客-CSDN博客](https://blog.csdn.net/weixin_44626319/article/details/119581767?spm=1001.2014.3001.5502)

同一局部空间内 defer 以堆栈方式存取，因此后声明的 defer 先执行

Defer 在声明时立刻对调用的参数进行求值，但函数调用直到周围的函数返回才执行。

Return 的过程可以被分解为以下三步：
1. 返回值赋值
2. 执行 defer 语句
3. 将结果返回
因此若当前函数声明中命名了返回值变量，可以在 defer 函数中对该返回值进行修改

### Select

对于 select 语句，在进入该语句时，会按源码的顺序对每一个 case 子句进行求值：这个求值只针对发送或接收操作的额外表达式

## 错误

### error

### panic

程序执行的异常会触发panic，**panic触发后立即执行并仅执行在该goroutine中的defer函数**，随后程序崩溃，输出包括panic value和函数调用的堆栈跟踪信息的日志

### recover

可通过在defer函数中定义**recover()** 函数捕获panic阻止程序中断，当一个函数发生了panic之后，若在当前函数中没有recover，会一直向外层传递直到主函数，最终中止协程，如果在过程中遇到了recover则被捕获
recover只能捕获本协程内的panic

利用recover处理panic指令，defer 必须在 panic 之前被声明

- 如下情况recover捕获不住，主要原因是代码直接使用了throw，退出了运行时
    - **内存溢出**：通过make申请大量内存，其本质是执行了mmap命令，其内存不够直接内部throw，抛出错误
    - **map并发读写**：go这里会导致直接无法捕获，其解释是go的设计希望不要编译好之后在运行时检测，而是要在--race条件下编译
    - **栈内存耗尽**：在栈的扩张中，会校验新的栈大小是否超过阈值 `1 << 20`
    - **尝试将 nil 函数交给 goroutine 启动**
    - **所有线程都休眠**：死锁
    - **重复解锁互斥锁**
- 可以捕获的异常
    - **数组 ( slice ) 下标越界**
    - **空指针异常**
    - **往已经 close 的 chan 中发送数据**
    - **类型断言**