---
{"dg-publish":true,"permalink":"/Code/6.Go/Go.1a.Go初始化/","title":"Go初始化","noteIcon":""}
---


# Go初始化

程序的初始化由被导入的最深层包开始逐层向外
每个包内部的初始化为**变量初始化->init()->main()**

## 变量初始化

函数作用域内的局部变量初始化顺序：**从左到右、从上到下**

package作用域变量：在每一个初始化周期，运行时(runtime)挑选一个没有任何依赖的变量初始化，该过程一直持续到所有的变量均被初始化或者出现依赖嵌套的情形

## init函数

> [!cite] 参考
> [init functions in Go.](https://medium.com/golangspec/init-functions-in-go-eac191b3860a)

- init函数先于main函数自动执行，不能被其他函数调用
- init函数没有输入参数、返回值
- 每个包可以有多个init函数；
- **包的每个源文件也可以有多个init函数**，由上至下执行
- 同一个包的init执行顺序，golang没有明确定义，编程时要注意程序不要依赖这个执行顺序
- 不同包的init函数按照包导入的依赖关系决定执行顺序

golang对没有使用的导入包会编译报错，使用`import_ packageName`以仅调用包的init函数

init函数常用于初始化无法以表达式方式初始化的package作用域变量
