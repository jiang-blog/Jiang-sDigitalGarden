---
{"dg-publish":true,"permalink":"/Code/6.Go/Go.1.Go原理/","title":"Go原理","noteIcon":""}
---


# Go原理

[[Code/6.Go/Go.1a.Go垃圾回收\|Go.1a.Go垃圾回收]]
[[Code/6.Go/Go.1b.Go并发\|Go.1b.Go并发]]
[[Code/6.Go/Go.1c.Go内存管理\|Go.1c.Go内存管理]]
[[Code/6.Go/Go.1d.Go编译优化\|Go.1d.Go编译优化]]

Go 程序的执行由两层组成：Go Program，Runtime，即用户程序和运行时
二者之间通过函数调用来实现内存管理、channel 通信、goroutines 创建等功能
用户程序进行的系统调用都会被 Runtime 拦截，以此来帮助它进行调度以及垃圾回收相关的工作

> ![5.png (860×764) (golang.design)](https://golang.design/go-questions/sched/assets/5.png)

