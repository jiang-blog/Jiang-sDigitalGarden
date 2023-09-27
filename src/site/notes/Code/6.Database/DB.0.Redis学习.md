---
{"dg-publish":true,"permalink":"/Code/6.Database/DB.0.Redis学习/","title":"Redis 学习","noteIcon":""}
---


# Redis 学习

Redis是一个是一个开源的使用 ANSI C 语言编写的高性能的 key-value 数据库

Redis属于NoSQL，也就是非关系型数据库

NoSQl的BASE理论是CAP中一致性的妥协。和传统事务的ACID截然不同，BASE不追求强一致性，而是允许数据在一段时间内是不一致的，但最终达到一致状态，从而获得更高的可用性和性能。

CAP 理论作为分布式系统的基础理论，描述的是一个分布式系统在以下三个特性中最多满足其中的两个特性
- 一致性(Consistency)
- 可用性(Availability)
- 分区容错性(Partition Tolerance)

[分布式系统的 CAP 定理与 BASE 理论-51CTO.COM](https://www.51cto.com/article/649412.html)

**Redis 高速原因**：
- Redis 的大部分操作在内存上完成
- Redis 的数据结构高效
- Redis 是单线程的，省去了很多线程上下文切换的时间和消耗
- Redis 采用了多路复用机制，使其在网络 IO 操作中能并发处理大量的客户端请求，实现高吞吐量

## [[Code/6.Database/DB.0a.Redis基础数据结构\|数据结构]]

## [[Code/6.Database/DB.0b.Redis原理\|原理]]
