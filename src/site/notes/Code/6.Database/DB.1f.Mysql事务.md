---
{"dg-publish":true,"permalink":"/Code/6.Database/DB.1f.Mysql事务/","title":"Mysql 事务","noteIcon":""}
---


# Mysql 事务

> [深入学习MySQL事务：ACID特性的实现原理 - 编程迷思 - 博客园 (cnblogs.com)](https://www.cnblogs.com/kismetv/p/10331633.html)

事务(Transaction)是访问和更新数据库的程序执行单元，事务中可能包含一个或多个sql语句，**这些语句要么都执行，要么都不执行**

```mysql
-- start transaction; -- 事务开始
begin;            -- 事务开始
... # n条sql语句
commit;            -- 提交事务，将结果写入数据库
```

如果sql语句执行出现问题，事务调用rollback回滚所有已经执行成功的sql语句

Mysql 中默认采用 *autocommit* 模式，如果没有 `start transaction` 或 `begin` 显式开启一个事务，则**每个 sql 语句都被当作一个事务执行**
如关闭autocommit，则所有sql语句都在同一事务中，直到执行commit或rollback结束该事务并开启一个新事务

此外存在一些特殊命令，在事务中执行这些命令时会马上强制执行 commit 提交事务，如 DDL 语句(create table/drop table/alter table)和lock tables 语句等等

## ACID

ACID是原子性、一致性、隔离性和持久性的缩写

### 原子性

**一个事务只有执行成功和失败回滚两种状态，没有中间态**

当事务对数据库进行修改时，InnoDB 会生成对应的 **undo log**，记录 sql 执行的相关信息
当发生回滚时 InnoDB 根据 undo log 内容做相反工作

### 持久性

**事务一旦提交，它对数据库的改变就应该是永久性的**

使用 redo log 保证 buffer pool 中的数据不随 Mysql 非正常停止而消失

### 隔离性

**事务内部的操作与其他事务是隔离的，并发执行的各个事务之间不能互相干扰**

通过锁机制和MVCC保证

### 一致性

**事务执行结束后，数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态**

数据库的完整性约束包括实体完整性(如行的主键存在且唯一)、列完整性(如字段的类型、大小、长度要符合要求)、外键约束、用户自定义完整性(如转账前后，两个账户余额的和应该不变)等

## 并发安全

### 并发问题

Mysql支持多个客户端连接，在同时处理多个事务的时候会出现并发问题，按照严重性从高到低分别为：
- **幻读**：在一个事务内前后查询的符合某条件的记录数量不相同
- **不可重复读**：在一个事务内前后读取的同一个数据不相同
- **脏读**：事务读取了另一个事务修改过但未提交的数据

### SQL隔离级别

SQL标准针对并发事务执行提出了四种隔离级别，隔离级别越高，开销越大，出现的问题越少，从低到高分别为：
- **读未提交**：指一个事务还没提交时，它做的变更就能被其他事务看到
  - 直接读取最新数据
  - 可能发生脏读、不可重复读和幻读
- **读已提交**：指一个事务提交之后，它做的变更才能被其他事务看到
  - 在每个语句执行前创建 Read View(数据快照)
  - 可能发生不可重复读和幻读
- **可重复读**：指一个事务执行过程中看到的数据，一直跟这个事务启动时看到的数据是一致的
  - 在事务启动时创建 Read View(数据快照)
  - 可能发生幻读
- **串行化**：会对记录加上读写锁，在多个事务对这条记录进行读写操作时，如果发生了读写冲突的时候，后访问的事务必须等前一个事务执行完成才能继续执行
  - 加读写锁
  - 三种问题都不会发生

> [!Note] Read View
> Read View 中包含四个重要字段：
> ![readview结构.drawio.png (900×437) (xiaolincoding.com)|600](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/readview%E7%BB%93%E6%9E%84.drawio.png){ #readview}


> [!attention] 事务启动时机
> `begin;/start transaction;`：事务开始未启动，事务启动于下一条增删查改语句
> `start transaction with consistent snapshot;`：事务立刻启动

### MVCC

MVCC 即**多版本并发控制**，通过比较读取行记录的 `trx_id` 与 Read View 确认记录版本，**通过版本链控制并发事务访问** 

事务更新行记录后 Mysql 记录相应的 undo log，通过 [[Code/6.Database/DB.1c.InnoDB数据管理#隐藏字段\|行记录隐藏字段]]中的 roll_pointer 串联 undo log 形成版本链，并将最新记录的 trx_id 设为当前事务 id

- `trx_id < min_trx_id:` 该版本记录创建/更新于当前事务启动前，可见
- `trx_id >= max_trx_id`: 该版本记录可能创建/更新于当前事务启动后，不可见
- `min_trx_id <= trx_id < max_trx_id`
  - `trx_id` 在 `m_ids` 中：该版本记录可能创建/更新于当前事务启动后，不可见
  - `trx_id` 不在 `m_ids` 中：该版本记录创建/更新于当前事务启动前，可见

只有在聚簇索引记录中才有 `trx_id` 和 `roll_pointer`，如果某个查询语句使用二级索引来查询，要使用下面的方式判断可见性：
二级索引树索引页的 [[Code/6.Database/DB.1c.InnoDB数据管理#页结构\|Page Header]] 部分存在 `PAGE_MAX_TRX_ID` 属性，记录修改该索引页的最大事务 id
执行增删改操作时，如果执行该操作的事物的事务 id 大于 `PAGE_MAX_TRX_ID`，则将 `PAGE_MAX_TRX_ID` 设置为执行操作的事务 id

当 SELECT 语句访问某个二级索引记录时，如果 readView 的 `min_trx_id > PAGE_MAX_TRX_ID`，说明该页面中的所有记录对该 ReadView 可见
否则需要执行回表判断，利用二级索引记录中的主键值进行回表操作，得到对应的聚簇索引记录后再按照聚簇索引的方式判断该可见性

### InnoDB隔离

**InnoDB 引擎的默认隔离级别为可重复读**
MySQL 里除了普通select是**快照读**，其他都是**当前读**，比如 update、insert、delete以及select … for update 等语句

InnoDB通过MVCC解决快照读下的幻读，通过[[Code/6.Database/DB.1.Mysql学习#行级加锁\|行级加锁]]解决当前读情况下的幻读，两种方法很大程度上避免了幻读，但是[**没能完全解决幻读**](https://xiaolincoding.com/mysql/transaction/phantom.html#%E5%B9%BB%E8%AF%BB%E8%A2%AB%E5%AE%8C%E5%85%A8%E8%A7%A3%E5%86%B3%E4%BA%86%E5%90%97)
>避免特殊场景下发生幻读的现象的方法就是**尽量在开启事务之后，马上执行 `select … for update` 这类当前读的语句**，因为它会对记录加 next-key lock，从而避免其他事务插入一条新记录
