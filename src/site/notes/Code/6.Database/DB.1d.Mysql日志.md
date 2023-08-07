---
{"dg-publish":true,"permalink":"/Code/6.Database/DB.1d.Mysql日志/","title":"Mysql日志","noteIcon":""}
---


# Mysql日志

## undo log 回滚日志

实现了事务中的*原子性*，主要用于**事务回滚和 MVCC**

> ![回滚事务.png (352×571) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E5%9B%9E%E6%BB%9A%E4%BA%8B%E5%8A%A1.png)

### 内容

Undo log 记录 sql 执行相关信息，每当 InnoDB 引擎对一条记录进行操作(修改、删除、新增)时，要把回滚时需要的信息都记录到 undo log 里，比如：
- **插入**一条记录时，存储记录主键值，回滚时把对应记录**删掉**(记录头信息中 delete_mask 标记)
- **删除**一条记录时，存储记录备份，回滚时把备份记录**插入表中**
- **更新**一条记录时，存储字段旧值，回滚时把这些列**更新为旧值**

发生回滚时引擎读取undo log里的数据，执行相反操作

一条记录的每一次更新操作产生的 undo log 格式都有一个 roll_pointer 指针和一个 trx_id 事务id：
- 通过 trx_id 可以知道该记录是被哪个事务修改的
- 通过 roll_pointer 指针可以将这些 undo log 串成一个链表，这个链表就被称为*版本链*

### 作用

undo log 两大作用：
- **实现事务回滚，保障事务的原子性**：事务处理过程中，如果出现了错误或者用户执行了 ROLLBACK 语句，MySQL 可以利用 undo log 中的历史数据将数据恢复到事务开始之前的状态。
- **实现 MVCC(多版本并发控制)关键因素之一**：MVCC 是通过 ReadView + undo log 实现的，undo log 为每条行记录保存多份历史数据，MySQL 在执行快照读(普通 select 语句)时会根据事务的 Read View 里的信息，顺着 undo log 的版本链找到满足其可见性的记录

### 刷盘

Undo log 和数据页一样，都通过 redo log 保证持久化，buffer pool 中有 undo 页，对 undo 页的修改也都会记录到 redo log

Undo log的删除是异步的，在它的使命结束之后，会由 Purge 线程删除

Purge 线程判断 Undo log 中的 trx_no 属性的值小于数据库中当前最早创建的 [[Code/6.Database/DB.1.Mysql学习#^readview\|ReadView]] 的 `m_low_limit_no` 属性的值，该 Undo 日志就可以删除

## redo log 重做日志

> ![wal.png (1292×977) (xiaolincoding.com)|600](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/wal.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

### 内容

redo log 是物理日志，记录数据页的修改，比如**对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新**，每当执行一个事务就会产生这样的一条或者多条物理日志

写入 redo log 的方式使用了追加操作， 所以磁盘操作是**顺序写**，而写入数据需要先找到写入位置，然后才写到磁盘，所以磁盘操作是**随机写**，前者开销远小于后者

### 作用

InnoDB 使用内存中的 Buffer pool 缓冲池提高数据库的读写性能，但可能丢失未写入磁盘的信息，redo log 保证了 Buffer pool 的持久性

### 刷盘

**redo log 也并非直接写入磁盘**，而是首先被写入缓存*redo log buffer*，之后再写入磁盘
redo log 刷盘时机：
- Mysql 正常关闭
- redo log buffer使用空间超过一半
- 后台线程每1s刷一次
- 每次事务提交时

InnoDB使用参数`innodb_flush_log_at_trx_commit`控制redo log的刷盘策略：
- 0：每次事务提交时将 redo log 留在 redo log buffer 中
- 1：每次事务提交时将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘
- 2：每次事务提交时，只将缓存在 redo log buffer 里的 redo log写到 redo log 文件(操作系统中的Page Cache)

> ![innodb_flush_log_at_trx_commit2.drawio.png (1061×863) (xiaolincoding.com)|600](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/innodb_flush_log_at_trx_commit2.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

**溢出控制**：
默认情况下 InnoDb 存在 redo log group，内含两个 redo log 文件，以**循环写**方式工作，相当于一个环形

InnoDB 用 write pos 表示 redo log 当前记录写到的位置，用 check point 表示当前要擦除的位置
当 write pos 追上 check point 时代表 redo log buffer 已满，此时 MySQL 被阻塞，不能再执行新的更新操作
之后 Mysql 将 Buffer Pool 中的脏页刷新到磁盘中，然后标记 redo log 中可擦除的记录，接着对旧的 redo log 记录进行擦除
等擦除完旧记录腾出了空间，checkpoint 往前移动，MySQL 恢复正常运行，继续执行新的更新操作

> ![checkpoint.png (1362×906) (xiaolincoding.com)|600](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/checkpoint.png)

## binlog 归档日志

### 内容

MySQL 在完成一条更新操作后，Server 层会生成一条 binlog，等事务提交时会将该事务执行过程中产生的所有 binlog 统一写入binlog 文件

binlog 文件是以**二进制形式**记录了所有数据库表**结构变更和表数据修改**的日志，**不会记录查询类的操作**，比如 SELECT 和 SHOW 操作

binlog 有 3 种格式类型：
- *STATEMENT*：每一条修改数据的 SQL 都会被记录到 binlog 中(逻辑日志)，但存在动态函数主从库执行结果不一致的问题
- *ROW*：记录行数据最终的修改值，缺点是每行数据的变化结果都会被记录，使 binlog 文件过大
- *MIXED*：包含了 STATEMENT 和 ROW 模式，根据不同的情况自动使用 ROW 模式和 STATEMENT 模式

### 刷盘

事务执行过程中，先把日志写到 binlog cache(Server 层)，**事务提交的时候**把 binlog cache 写到 binlog 文件中，一个事务的 binlog 是不能被拆开的，因此无论这个事务有多大(比如有很多条语句)，也要保证一次性写入

> ![binlogcache.drawio.png (721×461) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/binlogcache.drawio.png)

Mysql中每个线程都有binlog cache，通过参数`binlog_cache_size`控制大小，通过参数`sync_binlog`控制刷盘频率
- Sync_binlog = 0(default)：每次提交事务只 write 不 fsync，由操作系统决定刷盘时机
- Sync_binlog = N：每次提交事务write，提交N个事务刷盘一次

### 主从复制

Mysql 的主从复制依赖于binlog
>  ![主从复制过程.drawio.png (991×401) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E8%BF%87%E7%A8%8B.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

Mysql中的主从复制一般是**异步**的，通常分为三个阶段：
- *写入 Binlog*：主库写 binlog 日志，提交事务，并更新本地存储数据
- *同步 Binlog*：把 binlog 复制到所有从库上，每个从库把 binlog 写到relay log中
- *回放 Binlog*：从库读取relay log，回放 binlog，并更新存储引擎中的数据

从库数量过多会导致所需线程数过多，开销较大
实际使用中，一个主库一般跟 2～3 个从库(1 套数据库，1 主 2 从 1 备主)，这就是一主多从的 *MySQL 集群结构*

主从复制类型：
- **同步复制**：MySQL 主库提交事务的线程要等待所有从库的复制成功响应，才返回客户端结果。这种方式在实际项目中，基本上没法用，原因有两个：一是性能很差，因为要复制到所有节点才返回响应；二是可用性也很差，主库和所有从库任何一个数据库出问题，都会影响业务
- **异步复制**(默认模型)：MySQL 主库提交事务的线程并不会等待 binlog 同步到各从库，就返回客户端结果，这种模式一旦主库宕机，数据就会发生丢失
- **半同步复制**：MySQL 5.7 版本之后增加的一种复制方式，介于两者之间，事务线程不用等待所有的从库复制成功响应，只要一部分复制成功响应回来就行，比如一主二从的集群，只要数据成功复制到任意一个从库上，主库的事务线程就可以返回给客户端。半同步复制的方式，兼顾了异步复制和同步复制的优点，**即使出现主库宕机，至少还有一个从库有最新的数据，不存在数据丢失的风险**

## 两阶段提交

MySQL 使用*两阶段提交*来避免非正常情况下 redo log 和 binlog 逻辑不一致的问题

两阶段提交将单个事务的提交分为*准备*阶段和*提交*阶段，每个阶段由*协调者*和*参与者*共同完成

> [6 张图带你彻底搞懂分布式事务 XA 模式](https://developer.aliyun.com/article/783796)

当客户端执行 commit 语句或者在自动提交的情况下，MySQL 内部开启一个 XA 事务，分两阶段来完成 XA 事务的提交：

> ![两阶段提交.drawio.png (1157×842) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%A4%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4.drawio.png)

- **prepare 阶段**：将 XID(内部 XA 事务的 ID) 写入到 redo log，同时将 redo log 对应的事务状态设置为 prepare，然后将 redo log 持久化到磁盘(`innodb_flush_log_at_trx_commit = 1`)
- **commit 阶段**：把 XID 写入到 binlog，然后将 binlog 持久化到磁盘(`sync_binlog = 1`)，接着调用引擎的提交事务接口，将 redo log 状态设置为 commit，此时该状态并不需要持久化到磁盘，只需要 write 到文件系统的 page cache 中就够了，因为只要 binlog 写磁盘成功，就算 redo log 的状态还是 prepare 也没有关系，一样会被认为事务已经执行成功

两阶段提交**以 binlog 写成功为事务提交成功的标识**

### 问题

- **磁盘 I/O 次数高**：对于“双1”配置，每个事务提交都会进行两次 fsync(刷盘)，一次是 redo log 刷盘，另一次是 binlog 刷盘
- **锁竞争激烈**：两阶段提交虽然能够保证「单事务」两个日志的内容一致，但在「多事务」的情况下，却不能保证两者的提交顺序一致，因此，在两阶段提交的流程基础上，还需要加一个锁来保证提交的原子性，从而保证多事务的情况下，两个日志的提交顺序一致

### 组提交

MySQL 引入了 *binlog 组提交*(group commit)机制，当有多个事务提交的时候，会将多个 binlog 刷盘操作合并成一个，从而减少磁盘 I/O 的次数

引入了组提交机制后，prepare 阶段不变，只针对 commit 阶段，将 commit 阶段拆分为三个阶段：
- *flush* 阶段：多个事务按进入的顺序将 binlog 从 cache 写入文件(不刷盘)
  - 用于支持 redo log 的组提交
- *sync* 阶段：对 binlog 文件做 fsync 操作(多个事务的 binlog 合并一次刷盘)
  - 用于支持 binlog 的组提交
- *commit* 阶段：各个事务按顺序进行 InnoDB commit 操作
  - 承接 sync 阶段的事务，完成最后的引擎提交，使得 sync 可以尽早的处理下一组事务，最大化组提交的效率

**每个阶段都有一个队列且都有锁进行保护**，因此保证了事务写入的顺序，第一个进入队列的事务会成为 *leader*，leader 领导所在队列的所有事务，全权负责整队的操作，完成后通知队内其他事务操作结束

V5.6 中每个事务各自执行 prepare 阶段，也就是各自将 redo log 刷盘，这样就没办法对 redo log 进行组提交
V5.7 版本后，在 prepare 阶段不再让事务各自执行 redo log 刷盘操作，而是推迟到组提交的 flush 阶段，也就是说 prepare 阶段融合在了 flush 阶段，将 redo log 的刷盘延迟到了 flush 阶段之中，sync 阶段之前。通过延迟写 redo log 的方式，为 redolog 做了一次组写入，这样 binlog 和 redo log 都进行了优化

## 差异

- **undo log(回滚日志)**：是 Innodb 存储引擎层生成的日志，实现了事务中的**原子性**，主要**用于事务回滚和 MVCC**
- **redo log(重做日志)**：是 Innodb 存储引擎层生成的日志，实现了事务中的**持久性**，主要**用于掉电等故障恢复**
- **binlog (归档日志)**：是 Server 层生成的日志，主要**用于数据备份和主从复制**

redo log VS binlog：
- 生成对象不同：redo log 由InnoDB实现，binlog由Mysql的Sever层实现
- 内容不同：redo log 是物理日志，内容基于数据页，只记录未被刷入磁盘的buffer pool数据日志；binlog为二进制，可以基于sql语句，数据本身或二者混合，保存全量日志
- 写入方式不同：binlog为追加写，在事务提交时写入；redo log为循环写，有多种提交时机
- 作用不同：redo log 用于崩溃恢复，binlog 用于备份恢复和主从复制

redo log VS undo log：
- redo log 记录了此次事务*完成后*的数据状态，记录的是更新**之后**的值
- undo log 记录了此次事务*开始前*的数据状态，记录的是更新**之前**的值
