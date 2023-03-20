---
{"dg-publish":true,"permalink":"/Code/7.Database/Database.1.Mysql学习/","title":"Mysql学习","noteIcon":""}
---


# Mysql学习

## 基础知识

### 数据类型

> [MySQL 数据类型 | 菜鸟教程 (runoob.com)](https://www.runoob.com/mysql/mysql-data-types.html)

varchar(n)中n在V5.0以前代表最多存储字节数,之后代表**最多存储字符数**

### 操作分类

> [SQL |DDL、DQL、DML、DCL 和 TCL 命令 - GeeksforGeeks](https://www.geeksforgeeks.org/sql-ddl-dql-dml-dcl-tcl-commands/)

#### DDL

数据定义Data Definition Language，用于创建、修改和删除数据库结构而非数据
- CREATE：此命令用于创建数据库或其对象(如表、索引、函数、视图、存储过程和触发器)
- DROP：此命令用于从数据库中删除对象
- ALTER：这用于改变数据库的结构
- TRUNCATE：这用于从表中删除所有记录，包括为删除的记录分配的所有空间
- COMMENT：这用于向数据字典添加注释
- RENAME：这用于重命名数据库中存在的对象

#### DQL

数据查询Data Query Language，用于查询架构对象中的数据
- SELECT：用于从数据库中检索数据

#### DML

数据操作Data Manipulation Language，为用户提供添加、删除、更新数据的能力
- INSERT：用于将数据插入表中
- UPDATE：它用于更新表中的现有数据
- DELETE：用于从数据库表中删除记录
- LOCK：表并发控制
- CALL：调用一个 PL/SQL 或 JAVA 子程序
- EXPLAIN PLAN：描述数据的访问路径

#### DCL

数据控制Data Control Language，主要处理数据库系统的权限、权限和其他控制
- GRANT：此命令授予用户访问数据库的权限
- REVOKE：此命令撤消使用 GRANT 命令授予的用户访问权限

#### TCL

事务控制Transaction Control Language，事务将一组任务分组到单个执行单元中，每个事务都以特定任务开始，并在组中的所有任务成功完成时结束，如果任何任务失败，事务将失败
- BEGIN：开启事务
- COMMIT：提交事务
- ROLLBACK：在发生任何错误的情况下回滚事务
- SAVEPOINT：在事务中设置一个保存点
- SET TRANSACTION： 指定事务的特征

### 三范式

**第一范式1NF**: 数据库表的每一列都是不可分割的基本数据项，同一列中不能有多个值，即实体中的某个属性不能有多个值或者不能有重复的属性
在任何一个关系数据库中，1NF是对关系模式的基本要求，不满足1NF的数据库就不是关系数据库

**第二范式2NF**: 数据库表中的每个实例或行必须可以被唯一的区分
2NF建立在1NF之上，要求实体的属性完全依赖于主关键字不能部分依赖

**第三范式3NF**: 一个数据库表中不包含已在其它表中已包含的非主关键字信息
3NF建立在2NF之上，

**逆范式**: 通过增加冗余或重复的数据来提高数据库的读性能

### 主键&辅助键&外键

> [主键和外键约束 - SQL Server | Microsoft Learn](https://learn.microsoft.com/zh-cn/sql/relational-databases/tables/primary-and-foreign-key-constraints?view=sql-server-ver16)

主键(PRIMARY KEY)是用于唯一标识数据库中每条记录的字段，不能包含NULL值
选取主键的一个基本原则是：**不使用任何业务相关的字段作为主键**

**一个表只能有一个主键约束**，该约束可以包含一个或多个字段，构成主键的多个字段被统一称为联合主键
设置主键：
- CREATE创建库时指定
- 通过ALTER添加

主键设计：
- 自增主键设计：数据库会在插入数据时自动为每一条记录分配一个自增整数
  - 用 BIGINT 而非 INT做主键
  - 当达到自增整型类型的上限值时，再次自增插入，MySQL 数据库会报重复错误
  - MySQL 8.0 版本前自增值存在回溯现象，不建议使用
  - 自增值做主键，只能在当前实例中保证唯一，**不能保证全局唯一**
- UUID主键设计：使用一种全局唯一的字符串作为主键
- 业务自定义生成主键：在随机主键基础上结合业务信息生成主键

辅助键显示每条记录唯一的辅助值，它可用于标识记录，并且通常被索引，它也被称为**备用键**，**一个表可以有多个辅助键**

外键 (FOREIGN KEY) 是用于在两个表中的数据之间建立和加强链接的一列或多列的组合，约束两个表中数据的一致性和完整性
相关联字段中主键所在的表就是主表(父表)，外键所在的表就是从表(子表)

## 运行原理

### 架构

>![mysql查询流程.png (1261×721) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/sql%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B/mysql%E6%9F%A5%E8%AF%A2%E6%B5%81%E7%A8%8B.png)

Mysql的架构分为**Server 层**和**存储引擎层**

#### Server层

Server层包括连接器、查询缓存、分析器、优化器、执行器等，涵盖MySQL的大多数核心服务功能，以及所有的内置函数(如日期、时间、数学和加密函数等)，所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等
- 连接器负责与客户端建立连接、获取权限、维持和管理连接
- 查询缓存会根据查询请求在缓存中查找之前是否执行过相同的查询语句，如果有则直接返回结果
- 分析器会对SQL语句进行词法分析和语法分析
- 优化器负责决定如何执行SQL语句，选择最优的执行方案
- 执行器根据优化器生成的执行计划来执行SQL语句

#### 存储引擎层

存储引擎层负责数据的存储和提取。其架构设计允许用户根据性能、特性和其他需求来选择不同的数据存储方式
- InnoDB：Mysql默认和最通用的存储引擎，是一个事务安全(ACID兼容)的MySQL存储引擎，并且还提供了行级锁和外键的约束
- MyISAM：不支持事务，但是查询速度快，适合于查询比较频繁，插入、修改比较少的场景
- MEMORY(HEAP)：将表中的数据存储在内存中，查询速度快，但是不支持事务，适合于临时表的存储
- BDB(BerkeleyDB)：支持事务，但是查询速度较慢，适合于高并发的场景
- ARCHIVE：只支持插入和查询，不支持修改和删除，适合于数据归档

Mysql中可为不同表格设定特定存储引擎

### SQL语句执行

#### 连接数据库

```bash
mysql -h$ip -u$user -p
```

Mysql基于TCP进行传输，连接需要经过TCP三次握手，断开连接经过TCP四次挥手，连接后进行身份验证，通过则可以开始执行指令

Mysql 定义了空闲连接的**最大空闲时长**，由 `wait_timeout` 参数控制的，默认值是 8 h(28880s)，如果空闲连接超过了这个时间，连接器就会自动将它断开

Mysql 服务支持的**最大连接数**由`max_connections`参数控制，当前版本(V8.0.32)默认为151

Mysql连接根据是否执行单个sql指令分为短连接和长连接
长连接可减少连接和断连的开销，但可能导致占用内存增加，过大内存会导致系统强行杀死进程，可定期断开长连接或者客户端通过`mysql_reset_connection()`主动将连接恢复至刚创建的状态

#### 查询缓存(已废弃)

V8.0前Mysql在收到select语句后首先在Sever层的查询缓存中查找缓存的历史命令和执行结果，有则直接返回value，无则向下执行，获得结果后缓存
一个表的查询缓存会在表更新时清空，易导致缓存浪费，V8.0将查询缓存功能删除，V8.0以前可将参数 query_cache_type 设置成 DEMAND关闭查询缓存

#### 解析SQL

SQL语句在执行前先由*解析器*进行**词法分析**和**语法分析**
词法分析识别SQL语句中的关键字并构建SQL语法树
语法分析根据语法规则判断SQL语句的正确性

#### 执行SQL

每条`SELECT` 查询语句流程主要可以分为下面这三个阶段：
- prepare 阶段，也就是预处理阶段
- optimize 阶段，也就是优化阶段
- execute 阶段，也就是执行阶段

*预处理器*检查 SQL 查询语句中的表或者字段是否存在，将 `select *`中的 `*`符号，扩展为表上的所有列

*优化器*负责确定SQL语句的执行方案
通过在查询语句前加入`EXPLAIN` 可输出语句的执行计划，其中key为`PRIMARY`代表主键索引，`NULL`代表全表扫描，此外也可能为普通索引名

*执行器*

查询：
- 主键索引
  1. 调用 `read_first_record`函数，因为优化器选择的访问类型为 const，函数指针被指向为 InnoDB 引擎**索引查询**的接口，把搜索条件交给存储引擎，让存储引擎**定位符合条件的第一条记录**
  2. 存储引擎通过主键索引的 B+ 树结构定位到符合条件的第一条记录
      - 如果记录不存在，向执行器上报记录找不到的错误，然后查询结束
      - 如果记录存在，将记录返回给执行器
  1. 执行器接收到数据后判断是否符合查询条件，符合发送给客户端，否则跳过该记录
  2. 执行器通过一个while循环查询数据，第二次循环时调用`read_record`函数，在访问类型为const情况下调用函数永远返回-1，执行器退出循环结束查询
- 全表扫描
  1. 调用 `read_first_record`函数，优化器选择all访问类型，函数指针被指向为 InnoDB 引擎**全扫描**的接口，把搜索条件交给存储引擎，让存储引擎**读取表中的第一条记录**
  2. 执行器接收到数据后判断是否符合查询条件，符合即发送给客户端，否则跳过该记录(客户端等待查询语句完成后统一显示所有记录)
  3. 执行器第二次循环第二次循环时调用`read_record`函数，在访问类型为all情况下该函数指针同样指向 InnoDB 引擎**全扫描**的接口，令存储引擎继续读取下一条记录，再由执行器判断是否符合条件进而是否发送
  4. 重复过程直到读完所有记录，存储引擎返回读取完毕信息
  5. 执行器退出循环，结束查询
- 索引下推(针对联合索引)：
`select * from t_user  where age > 20 and reward = 100000;`
  1. 执行器首先调用引擎的索引查询接口定位到满足查询(age>20)的第一条二级索引记录
  2. 存储引擎先不执行回表操作，先判断索引中包含的列(reward)是否满足条件(=10000)，仅当条件成立时执行回表操作，将记录返回执行器

更新：
1. 通过B+树搜索记录
    - 存在Buffer pool则直接将池中记录返回执行器
    - 否则先将记录所在数据页读入Buffer Pool
1. 确认更新前后记录是否相同
2. 开启事务，更新记录前记录undo log，undo log 会写入 Buffer Pool 中的 Undo 页面，同时记录redo log
3. InnoDB更新Buffer pool中记录，将相应数据页标为脏页，记录redo log
4. 后续由后台线程选择合适时机将脏页写入磁盘
5. 执行完成后记录该语句对应的binlog

## 数据存储

```bash
var/lib/mysql/$database_name # 数据库默认地址
db.opt # 存储当前数据库的默认字符集和字符校验规则
$table_name.frm # 表结构
$table_name.ibd # 表数据
```
表数据既可以放在共享表空间文件(ibdata1)里，也可以存放在独占表空间文件($table_name.ibd.ibd)，默认为后者

表空间由**段(segment)**、**区(extent)**、**页(page)**、**行(row)** 组成
>![表空间结构.drawio.png (686×651) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/%E8%A1%A8%E7%A9%BA%E9%97%B4%E7%BB%93%E6%9E%84.drawio.png)

行：数据库表中记录的存放方式，每行记录根据不同的行格式有不同的存储结构
页：InnoDB读取数据的基本单位，读取记录时以页为单位整体读入内存，**每页的默认大小为16KB**
区：在表中数据量大时，以区代替页为单位为索引分配空间，使得链表中相邻的页在磁盘中的物理位置相邻，便于顺序读取，**每个区的大小为1MB**
段：一般分为数据段，索引段，回滚段等
- 索引段：存放B+树的非叶子节点的区的集合
- 数据段：存放叶子节点的区的集合
- 回滚段：存放回滚数据的区的集合

### 行格式

InnoDB提供四种行格式，分别为Redundant、Compact、Dynamic和 Compressed
redundant为V5.0前使用的非紧凑行格式，V5.1后默认使用紧凑行格式compact，V5.7后默认使用改进紧凑行格式Dynamic

compact行格式：
> ![COMPACT.drawio.png (2336×562) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/COMPACT.drawio.png)

#### 额外信息

变长字段长度列表**只出现在数据表有变长字段的时候**，**逆序存放**记录中变长字段的真实数据占用的字节数

NULL值列表**只出现在数据表有字段未定义为NOT NULL的时候**，**逆序存放**以bitmap形式记录当前记录中的NULL值

记录头信息中指向下一个记录的指针指向的是下一条记录中额外信息和真实数据的分界处
逆序存放可以使得位置靠前的记录的真实数据和数据对应的字段长度信息同时存在于一个 CPU Cache Line 中，提高 CPU Cache 的命中率

记录头信息中包含:
- delete_mask：标识数据是否被删除
- next_record：下一条记录的位置
- record_type：当前记录的类型

#### 真实数据

MySQL 规定除了 TEXT、BLOBs 等大对象类型之外，一行记录中其他所有列(不包括隐藏字段和记录头信息)占用的字节长度加起来不能超过 65535 个字节，如果有多个字段的话，要保证**所有字段的长度 + 变长字段字节数列表所占用的字节数 + NULL值列表所占用的字节数 <= 65535**

因此varchar(n) 中 n的最大取值为
$$\frac{(65535-len(变长字段长度列表)-len(NULL值列表))}{max(单字符字节数)}$$

##### 隐藏字段

真实数据中包含三个隐藏字段：
- row_id：6bytes，当表没有主键或唯一约束列时InnoDB为记录添加row_id隐藏字段，row_id非必需
- trx_id：6bytes，事务id，表明数据的生成事务
- roll_pointer：7bytes，记录上一个版本的指针

#### 行溢出

Compressed 和 Dynamic 行格式和Compact的主要区别在于处理行溢出数据时

当一条记录的大小(<=65535bytes)大于一个页的大小(16kb)会发生行溢出，会使用额外的溢出页存储数据

Compact在记录的真实数据处仅保存一部分数据，然后用20字节指向溢出页地址
Compressed 和 Dynamic采用完全行溢出，真实数据处仅存储20字节指针，完全使用溢出页存放实际数据

### 数据页

> ![243b1466779a9e107ae3ef0155604a17.png (692×843) (xiaolincoding.com)](https://cdn.xiaolincoding.com//mysql/other/243b1466779a9e107ae3ef0155604a17.png)

文件头：页信息
页头：页状态信息
最大最小记录：数据页中存在两个虚拟记录，分别代表页中的最大索引记录和最小索引记录
用户记录：存储行记录
空闲空间：还未使用的空间
页目录：存储用户记录的相对位置，起索引作用
文件尾：校验页是否完整

文件头中存在指向前后数据页的指针，使得页集合构成一个双向链表

数据页中的记录按照主键顺序组成单向链表，便于插入删除，通过页目录的索引实现快速访问

> ![261011d237bec993821aa198b97ae8ce.png (1080×1228) (xiaolincoding.com)](https://cdn.xiaolincoding.com//mysql/other/261011d237bec993821aa198b97ae8ce.png)

数据页将未标记为已删除的记录分组存储，分组数量遵循原则：
- 第一个分组仅包含页内最小记录(虚拟)
- 最后一个分组记录条数为1-8条
- 其余分组记录条数为4-8条
每组的最后一条记录为最大记录，该记录头信息中存储该组的记录数量
页目录存储每组最后一条记录的地址偏移量,称之为槽
通过槽可以二分查找确定记录组，再通过顺序查找确定目标记录

### Buffer pool

> [揭开 Buffer Pool 的面纱 | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/mysql/buffer_pool/buffer_pool.html#buffer-pool-%E7%BC%93%E5%AD%98%E4%BB%80%E4%B9%88)
> [Innodb Buffer Pool详解 - 墨天轮 (modb.pro)](https://www.modb.pro/db/607938)
> [详解MySQL中的Buffer Pool，深入底层带你搞懂它！ - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1828772)
> [[玩转MySQL之十]InnoDB Buffer Pool详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/65811829)

InnoDB通过将磁盘中取出的数据缓存至内存的buffer pool中提升查询性能

Buffer pool默认配置为`128MB`，可以通过`innodb_buffer_pool_size` 参数设置 Buffer Pool 的大小，一般建议设置成可用物理内存的60%~80%

在Mysql服务启动时InnoDB为buffer pool申请一片连续的内存空间，按照默认`16kb`的大小划分为缓存页

#### 具体操作

- 读取数据时，如果数据存在于buffer pool中，客户端直接读取池中数据
- 修改数据时，首先修复buffer pool中数据所在页，然后将该页设置为脏页，通过后台进程将脏页写入磁盘

#### 内容

Buffer pool 中包含有*数据页*、*索引页*、*插入缓存页*、*undo页*、*自适应hash页*以及*锁信息页*

> ![freelist.drawio.png (1217×597) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/freelist.drawio.png)

每个缓存页对应存在一个**控制块**，内含缓存页的表空间、页号、缓存页地址、链表节点等信息

控制块和缓存页间的灰色部分称为碎片空间

空闲页的管理通过**free链表**实现，其将空闲缓存页的控制块作为双向链表的节点，头节点为特殊节点，包含链表的真正头节点地址，尾节点地址，以及当前链表中节点的数量等信息
脏页的管理通过**flush 链表**实现，与free链表结构相同

#### 数据管理-LRU

buffer pool使用最近最少使用(LRU)算法的变体将很少使用的数据从缓存中清除

简单的LRU为所有数据维护一个顺序链表，不断将新读数据页插入表头并删除表尾数据页，存在两大问题
- **预读失效**：程序具有空间局部性，Mysql在加载数据页时会预读加载相邻的数据页，简单的LRU算法会导致这些未访问的数据页占据链表前端
- **buffer pool 污染**：当某一个sql扫描大量数据时会将buffer pool里的所有数据都替换出去，导致大量热数据被淘汰

InnoDB使用的LRU变体将LRU划分为**young**和**old**两个区域，young 区域在 LRU 链表的前半部分，old 区域在后半部分
预读页最初被加入old 区域头部，被真正访问且已在old区域停留超过一定时间时才被插入young区域头部
old区域解决预读失效，时间限制解决pool污染
> ![young+old.png (842×212) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/young%2Bold.png)

old 区域占整体的比例通过 `innodb_old_blocks_pct` 参数设置，默认是 37
old区域时间限制通过`innodb_old_blocks_time` 参数设置，默认是 1000 ms

#### 写入磁盘

InnoDB通过先写redo log 日志的方式避免宕机造成的buffer pool数据丢失

触发写入：
- redo log日志满了将所有脏页写入
- buffer pool清除数据时先将数据中的脏页写入
- Mysql判定处于空闲状态时定期刷入适量脏页
- Mysql正常关闭前将所有脏页写入

redo log比直接将Buffer Pool中修改的数据写入磁盘(即刷脏)要快：
- 刷脏是随机IO，因为每次修改的数据位置随机，但写redo log是追加操作，属于顺序IO
- 刷脏是以数据页(Page)为单位的，MySQL默认页大小是16KB，一个Page上一个小修改都要整页写入；而redo log中只包含真正需要写入的部分，无效IO大大减少

### Change buffer

当更新一条记录，如果该记录的数据页并没有内存中，那么就需要从磁盘读取该数据页到内存，然后在更新数据页。可以看到这种场景下，会发生磁盘 io，如果更新的记录很多，然后这些记录的数据页都不在内存中的话，就可能需要发生很多次磁盘 io，这样还是挺影响更新操作的性能的。

那么为了提高更新操作的性能，innodb 存储引擎引入了 change buffer。当更新一条记录，如果该记录的数据页并没有内存中，就将更新操作缓存在 change buffer 中，这样就不需要从磁盘中读入这个数据页了。在下次查询需要访问这个数据页的时候，将数据页读入内存，然后将 change buffer 缓存的操作应用到这个数据页，这个过程就是 merge(合并)，通过这种方式就能保证这个数据逻辑的正确性。

不过 change buffer 只限于用在普通索引的场景下，而不适用于唯一索引。因为唯一索引有唯一性的约束，在更新记录的时候，需要将数据页读入内存，判断有没有冲突。

change buffer 虽然是在内存，但是 change buffer 新写入的内容也会写入到 redo log，从而保证 change buffer 持久性

## 事务

> [深入学习MySQL事务：ACID特性的实现原理 - 编程迷思 - 博客园 (cnblogs.com)](https://www.cnblogs.com/kismetv/p/10331633.html)

事务(Transaction)是访问和更新数据库的程序执行单元，事务中可能包含一个或多个sql语句，这些语句要么都执行，要么都不执行

```mysql
start transaction;
... # n条sql语句
commit;
```
start transaction标识事务开始，commit提交事务，将执行结果写入到数据库
如果sql语句执行出现问题，事务调用rollback回滚所有已经执行成功的sql语句

**Mysql中默认采用autocommit模式**：
如果没有`start transaction`显式开启一个事务，则每个sql语句都被当作一个事务执行

如关闭autocommit，则所有sql语句都在同一事务中，直到执行commit或rollback结束该事务并开启一个新事务

此外存在一些特殊命令，在事务中执行这些命令时会马上强制执行commit提交事务；如DDL语句(create table/drop table/alter/table)、lock tables语句等等

### ACID

ACID是原子性、一致性、隔离性和持久性的缩写
**原子性**指的是
**一致性**是说数据状态一定是一致的
**隔离性**是说事务之间互不影响
**持久性**是指事务执行完成后，对数据所做的操作持久的保存在数据库中

#### 原子性

一个事务只有执行成功和失败回滚两种状态，没有中间态

当事务对数据库进行修改时，InnoDB会生成对应的**undo log**，记录sql执行的相关信息
当发生回滚时InnoDB根据undo log内容做相反工作

#### 持久性

事务一旦提交，它对数据库的改变就应该是永久性的

使用redo log保证buffer pool中的数据不随My sql非正常停止而消失

#### 隔离性

事务内部的操作与其他事务是隔离的，并发执行的各个事务之间不能互相干扰

通过锁机制和MVCC保证

#### 一致性

事务执行结束后，数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态

数据库的完整性约束包括实体完整性(如行的主键存在且唯一)、列完整性(如字段的类型、大小、长度要符合要求)、外键约束、用户自定义完整性(如转账前后，两个账户余额的和应该不变)等

### 并发安全

Mysql支持多个客户端连接，在同时处理多个事务的时候会出现并发问题，按照严重性从高到低分别为：
- **幻读**：在一个事务内前后查询的符合某条件的记录数量不相同
- **不可重复读**：在一个事务内前后读取的同一个数据不相同
- **脏读**：事务读取了另一个事务修改过但未提交的数据

#### SQL隔离级别

SQL标准针对并发事务执行提出了四种隔离级别，隔离级别越高，开销越大，出现的问题越少，从低到高分别为：
- **读未提交**：指一个事务还没提交时，它做的变更就能被其他事务看到(可能发生脏读、不可重复读和幻读)
- **读已提交**：指一个事务提交之后，它做的变更才能被其他事务看到(可能发生不可重复读和幻读)
- **可重复读**：指一个事务执行过程中看到的数据，一直跟这个事务启动时看到的数据是一致的(可能发生幻读)
- **串行化**：会对记录加上读写锁，在多个事务对这条记录进行读写操作时，如果发生了读写冲突的时候，后访问的事务必须等前一个事务执行完成才能继续执行(三种问题都不会发生)

具体实现方法：
- 读未提交：直接读取最新数据
- 读已提交：在每个语句执行前创建Read View(数据快照)
- 可重复读：在事务启动时创建Read View(数据快照)
- 串行化：加读写锁

> [!attention] 事务启动时机
> `begin;/start transaction;`：事务开始未启动，事务启动于下一条增删查改语句
> `start transaction with consistent snapshot;`：事务立刻启动

##### Read View

Read View中包含四个重要字段：
> ![readview结构.drawio.png (900×437) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/readview%E7%BB%93%E6%9E%84.drawio.png)

#### MVCC

MVCC即**多版本并发控制**，通过比较读取行记录的trx_id与Read View确认记录版本，**通过版本链控制并发事务访问**

事务更新记录后Mysql记录相应的 undo log，通过行记录隐藏字段中的roll_pointer串联旧记录形成版本链，并将最新记录的trx_id设为当前事务id

- trx_id < min_trx_id: 该版本记录创建/更新于当前事务启动前，可见
- trx_id >= max_trx_id: 该版本记录可能创建/更新于当前事务启动后，不可见
- min_trx_id <= trx_id < max_trx_id
  - trx_id 在m_ids中：该版本记录可能创建/更新于当前事务启动后，不可见
  - trx_id 不在m_ids中：该版本记录创建/更新于当前事务启动前，可见

#### InnoDB隔离

**InnoDB 引擎的默认隔离级别为可重复读**
MySQL 里除了普通select是**快照读**，其他都是**当前读**，比如 update、insert、delete以及select … for update 等语句

InnoDB通过MVCC解决快照读下的幻读，通过[[Code/7.Database/Database.1.Mysql学习#行级加锁\|行级加锁]]解决当前读情况下的幻读，两种方法很大程度上避免了幻读，但是[**没有能完全解决幻读**](https://xiaolincoding.com/mysql/transaction/phantom.html#%E5%B9%BB%E8%AF%BB%E8%A2%AB%E5%AE%8C%E5%85%A8%E8%A7%A3%E5%86%B3%E4%BA%86%E5%90%97)
>避免特殊场景下发生幻读的现象的话，就是**尽量在开启事务之后，马上执行`select … for update` 这类当前读的语句**，因为它会对记录加 next-key lock，从而避免其他事务插入一条新记录

### 两阶段提交

MySQL 使用*两阶段提交*来避免非正常情况下redo log和binlog逻辑不一致的问题

两阶段提交将单个事务的提交分为*准备*阶段和*提交*阶段，每个阶段由*协调者*和*参与者*共同完成

>[6 张图带你彻底搞懂分布式事务 XA 模式](https://developer.aliyun.com/article/783796)

当客户端执行 commit 语句或者在自动提交的情况下，MySQL 内部开启一个 XA 事务，分两阶段来完成 XA 事务的提交：

>![两阶段提交.drawio.png (1157×842) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%A4%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

- **prepare 阶段**：将 XID(内部 XA 事务的 ID) 写入到 redo log，同时将 redo log 对应的事务状态设置为 prepare，然后将 redo log 持久化到磁盘(innodb_flush_log_at_trx_commit = 1)
- **commit 阶段**：把 XID 写入到 binlog，然后将 binlog 持久化到磁盘(sync_binlog = 1 )，接着调用引擎的提交事务接口，将 redo log 状态设置为 commit，此时该状态并不需要持久化到磁盘，只需要 write 到文件系统的 page cache 中就够了，因为只要 binlog 写磁盘成功，就算 redo log 的状态还是 prepare 也没有关系，一样会被认为事务已经执行成功

两阶段提交**以 binlog 写成功为事务提交成功的标识**

#### 问题

- **磁盘 I/O 次数高**：对于“双1”配置，每个事务提交都会进行两次 fsync(刷盘)，一次是 redo log 刷盘，另一次是 binlog 刷盘
- **锁竞争激烈**：两阶段提交虽然能够保证「单事务」两个日志的内容一致，但在「多事务」的情况下，却不能保证两者的提交顺序一致，因此，在两阶段提交的流程基础上，还需要加一个锁来保证提交的原子性，从而保证多事务的情况下，两个日志的提交顺序一致

#### 组提交

MySQL 引入了 binlog *组提交*(group commit)机制，当有多个事务提交的时候，会将多个 binlog 刷盘操作合并成一个，从而减少磁盘 I/O 的次数

引入了组提交机制后，prepare 阶段不变，只针对 commit 阶段，将 commit 阶段拆分为三个阶段：
- *flush* 阶段：多个事务按进入的顺序将 binlog 从 cache 写入文件(不刷盘)
  - 用于支持 redo log 的组提交
- *sync* 阶段：对 binlog 文件做 fsync 操作(多个事务的 binlog 合并一次刷盘)
  - 用于支持 binlog 的组提交
- *commit* 阶段：各个事务按顺序做 InnoDB commit 操作
  - 承接 sync 阶段的事务，完成最后的引擎提交，使得 sync 可以尽早的处理下一组事务，最大化组提交的效率

**每个阶段都有一个队列且都有锁进行保护**，因此保证了事务写入的顺序，第一个进入队列的事务会成为 *leader*，leader领导所在队列的所有事务，全权负责整队的操作，完成后通知队内其他事务操作结束

V5.6 的组提交逻辑中，每个事务各自执行 prepare 阶段，也就是各自将 redo log 刷盘，这样就没办法对 redo log 进行组提交
V5.7 版本中，在 prepare 阶段不再让事务各自执行 redo log 刷盘操作，而是推迟到组提交的 flush 阶段，也就是说 prepare 阶段融合在了 flush 阶段，将 redo log 的刷盘延迟到了 flush 阶段之中，sync 阶段之前。通过延迟写 redo log 的方式，为 redolog 做了一次组写入，这样 binlog 和 redo log 都进行了优化

## 锁

Mysql根据加锁的范围将锁分为**全局锁**、**表级锁**和**行级锁**

表锁和行锁满足读读共享、读写互斥、写写互斥

### 全局锁

全局加锁
```mysql
flush tables with read lock
```

该sql指令将数据库设为只读状态，任何其他线程执行的对数据的增删改操作以及对表结构的更改操作都被阻塞

释放全局锁
```mysql
unlock tables
```

**全局锁主要用于全库逻辑备份**，备份时间过长会造成业务停滞
通过创建事务在**可重复读**隔离级别下进行备份可避免该问题

### 表级锁

#### 表锁

```mysql
//表级别的共享锁，也就是读锁；
lock tables tablename read;
//表级别的独占锁，也就是写锁；
lock tables tablename write;
```
表锁同时限制本线程和其他线程对表的读写

#### 元数据锁MDL

MDL锁被Mysql隐式使用
MDL读锁
- 查询表结构时自动添加
- 执行查询语句时如需扫描表则自动添加
- 执行存储过程时如访问某表则自动添加
MDL写锁
- 修改表结构时自动添加
- 执行写操作时如需扫描表则自动添加
- 执行存储过程时如修改某表则自动添加

MDL锁在事务提交后被释放，且**MDL写锁获取优先级高于读锁**
当长事务MDL读锁堵塞某MDL写锁时,之后的所有MDL读锁都被堵塞

#### 意向锁

- 在使用 InnoDB 引擎的表里对某些记录加上*共享锁*之前，需要先在表级别加上一个*意向共享锁*
- 在使用 InnoDB 引擎的表里对某些记录加上*独占锁*之前，需要先在表级别加上一个*意向独占锁*

意向共享锁和意向独占锁只会和表锁发生冲突
意向锁的目的是快速判断表里是否有记录被加锁

#### AUTO-INC锁

AUTO-INC锁用于保护表内的自增值，在插入数据时先为表加一个AUTO-INC锁再为被 `AUTO_INCREMENT` 修饰的字段赋递增值，以保证连续递增

AUTO-INC锁采用特殊表锁机制，在一个SQL语句执行完后立刻释放，而不是在事务结束时释放

V5.1.22后InnoDB提供了一种轻量级新锁代替AUTO-INC锁，在插入数据时为自增字段加锁，赋值后立刻解锁

InnoDB 通过`innodb_autoinc_lock_mode`变量设置使用锁
但可能发生[主从不一致问题](https://xiaolincoding.com/mysql/lock/mysql_lock.html#auto-inc-%E9%94%81)

### 行级锁

InnoDB 引擎支持行级锁而 MyISAM 引擎并不支持

**InnoDB中行锁要等到事务结束时才释放**

#### Record Lock 记录锁

Record Lock分为S锁和X锁，S锁相当于读锁，X锁相当于写锁

#### Gap Lock 间隙锁

Gap Lock只存在于可重复读隔离级别，为范围加锁，范围为开开区间

间隙锁也分为S锁和X锁，但**间隙锁之间都相互兼容**，不存在互斥关系

锁定区间：根据检索条件向左寻找最靠近检索条件的记录值A，作为左区间，向右寻找最靠近检索条件的记录值B作为右区间，即锁定的间隙为(A，B)

#### Next-Key Lock 临键锁

临键锁是记录锁和间隙锁的组合，锁定索引本身以及索引之前的间隙，相当于左开右闭区间范围锁

锁定区间：
> ![64715c80d9b748d5b9208f244706040a.png (927×315) (alicdn.com)](https://ucc.alicdn.com/pic/developer-ecology/64715c80d9b748d5b9208f244706040a.png)

#### 插入意向锁

**插入意向锁是一种特殊的间隙锁**，但不同于间隙锁的是，该锁只用于并发插入操作

当执行插入操作时，总会检查当前插入位置的下一条记录(已存在的主索引节点)上是否存在间隙锁对象，判断是否间隙被锁定，如果锁住了，则判定和插入意向锁冲突，当前插入操作被阻塞
**插入意向锁配合间隙锁或临键锁一起防止了幻读操作**

插入意向锁之间不会互相冲突，多个插入操作同时插入同一个间隙时无需互相等待

> [!note]
> MySQL 加锁时，先生成锁结构，然后设置锁的状态，如果锁状态是等待状态，并不意味着事务成功获取到了锁，只有当锁状态为正常状态时，才代表事务成功获取到了锁

#### 隐式锁

插入语句正常执行时不生成锁结构，靠聚簇(主键)索引记录自带的 trx_id 隐藏列作为**隐式锁**来保护记录

隐式锁只有在特殊情况下才会转换为显示锁，例如插入操作存在唯一键冲突时，先插入的记录上的隐式锁转换为显式记录锁，阻止另一插入操作

### 行级加锁

普通的select语句不对记录加锁，属于快照读
在查询时加锁的语句称为锁定读
```mysql
//对读取的记录加共享锁(S型锁)
select ... lock in share mode;
//对读取的记录加独占锁(X型锁)
select ... for update;
```
update 和 delete操作都会加行级独占锁
```mysql
//对操作的记录加独占锁(X型锁)
update table .... where id = 1;
//对操作的记录加独占锁(X型锁)
delete from table where id = 1;
```

**行级锁加锁的对象是索引**，锁的基本单位是临键锁，但在**仅使用记录锁或间隙锁就能避免幻读**的情况下，临键锁退化为记录锁或间隙锁

唯一索引等值查询
- 索引上的等值查询，在给唯一索引加锁时，临键锁会退化为*记录锁*，因为主键是唯一的
- 索引上的等值查询， 继续向右遍历时且最后一个值不满足等值条件的时候，临键锁退化为*间隙锁*

非唯一索引等值查询：
- 非唯一索引等值查询的过程是一个扫描的过程，**对二级索引进行加锁分析时根据二级索引的B树(按照二级索引对记录进行排序)**，扫描到第一条不符合条件的二级索引记录就停止扫描
- 当查询的记录「存在」时，由于不是唯一索引，所以可能存在索引值相同的记录
  - 扫描的过程中对符合条件的二级索引记录加的是*临键锁*
  - 在符合查询条件的记录的主键索引上加*记录锁*
  - 对于第一个不符合条件的二级索引记录，该二级索引的临键锁会退化成*间隙锁*
- 当查询的记录「不存在」时
  - 第一条不符合条件的二级索引记录的临键锁会退化成*间隙锁*
  - 因为不存在满足查询条件的记录，所以不会对主键索引加锁

唯一索引范围查询：
- 针对`>=`的范围查询，等值查询对应的索引如存在，其上的临键锁退化为*记录锁*
- 针对`<`的范围查询，等值查询对应的索引如存在，其上的临键锁退化为*间隙锁*

非唯一索引范围查询过程中索引的临键锁不会退化为间隙锁和记录锁

没有使用索引的锁定读语句或update和delete语句会**导致全表扫描，对每一条记录都加*临键锁***

#### 阻塞

插入语句在插入一条记录之前，先定位到该记录在 B+树 的位置，如果插入的位置的下一条记录的索引上有间隙锁，才会发生阻塞

对于update语句的全局加锁，可将 MySQL 里的 `sql_safe_updates` 参数设置为 1，开启安全更新模式
该模式下update必须满足以下条件之一才能执行成功：
- 使用 where，并且 where 条件中必须有索引列
- 使用 limit；
- 同时使用 where 和 limit，此时 where 条件中可以没有索引列
delete 语句必须满足以下条件能执行成功：
- 同时使用 where 和 limit，此时 where 条件中可以没有索引列

### 死锁

死锁的四个必要条件：**互斥、占有且等待、不可强占用、循环等待**

Mysql有两种策略通过*打破循环等待条件*来解除死锁状态：
- 设置事务等待锁的超时时间，当一个事务的等待时间超过该值后进行回滚释放锁，InnoDB 中通过参数 `innodb_lock_wait_timeout` 设置超时时间，默认值为50s
- 开启主动死锁检测，主动死锁检测在发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行，通过参数 `innodb_deadlock_detect`设置，默认为on

## 日志

### undo log 回滚日志

实现了事务中的*原子性*，主要用于**事务回滚和 MVCC**

> ![回滚事务.png (352×571) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E5%9B%9E%E6%BB%9A%E4%BA%8B%E5%8A%A1.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

#### 内容

每当 InnoDB 引擎对一条记录进行操作(修改、删除、新增)时，要把回滚时需要的信息都记录到 undo log 里，比如：
- **插入**一条记录时，存储记录主键值，回滚时把对应记录**删掉**
- **删除**一条记录时，存储记录备份，回滚时把备份记录**插入**表中
- **更新**一条记录时，存储字段旧值，回滚时把这些列**更新为旧值**

发生回滚时引擎读取undo log里的数据，执行相反操作

一条记录的每一次更新操作产生的 undo log 格式都有一个 roll_pointer 指针和一个 trx_id 事务id：
- 通过 trx_id 可以知道该记录是被哪个事务修改的
- 通过 roll_pointer 指针可以将这些 undo log 串成一个链表，这个链表就被称为*版本链*

#### 作用

undo log 两大作用：
- **实现事务回滚，保障事务的原子性**。事务处理过程中，如果出现了错误或者用户执 行了 ROLLBACK 语句，MySQL 可以利用 undo log 中的历史数据将数据恢复到事务开始之前的状态。
- **实现 MVCC(多版本并发控制)关键因素之一**。MVCC 是通过 ReadView + undo log 实现的。undo log 为每条记录保存多份历史数据，MySQL 在执行快照读(普通 select 语句)的时候，会根据事务的 Read View 里的信息，顺着 undo log 的版本链找到满足其可见性的记录

#### 刷盘

undo log 和数据页的刷盘策略是一样的，都需要通过 redo log 保证持久化，buffer pool 中有 undo 页，对 undo 页的修改也都会记录到 redo log

### redo log 重做日志

> ![wal.png (1292×977) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/wal.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

#### 内容

redo log 是物理日志，记录数据页的修改，比如**对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新**，每当执行一个事务就会产生这样的一条或者多条物理日志

写入 redo log 的方式使用了追加操作， 所以磁盘操作是**顺序写**，而写入数据需要先找到写入位置，然后才写到磁盘，所以磁盘操作是**随机写**，前者开销远小于后者

#### 作用

InnoDB使用内存中的Buffer pool 缓冲池提高数据库的读写性能，但可能丢失未写入磁盘的信息
InnoDB使用redo log同步记录修改信息，在适当时候由后台线程将Buffer pool中脏页刷入磁盘，这种方法称为**WAL (Write-Ahead Logging)技术**

#### 刷盘

**redo log并非直接写入磁盘**，而是存在缓存*redo log buffer*，redo log 首先被写入buffer，之后再写入磁盘
redo log 刷盘时机：
- Mysql正常关闭
- redo log buffer使用空间超过一半
- 后台线程每1s刷一次
- 每次事务提交时

InnoDB使用参数`innodb_flush_log_at_trx_commit`控制redo log的刷盘策略：
- 0：每次事务提交时将 redo log 留在 redo log buffer 中
- 1：每次事务提交时将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘
- 2：每次事务提交时，只将缓存在 redo log buffer 里的 redo log写到 redo log 文件(操作系统中的Page Cache)

> ![innodb_flush_log_at_trx_commit2.drawio.png (1061×863) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/innodb_flush_log_at_trx_commit2.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

**溢出控制**：
默认情况下InnoDb存在redo log group，内含两个redo log文件，以**循环写**方式工作，相当于一个环形，InnoDB 用 write pos 表示 redo log 当前记录写到的位置，用 checkpoint 表示当前要擦除的位置

> ![checkpoint.png (1362×906) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/checkpoint.png)

>如果 write pos 追上了 checkpoint，意味着 redo log 文件满了，这时 MySQL 不能再执行新的更新操作，也就是说 MySQL 会被阻塞，此时会停下来将 Buffer Pool 中的脏页刷新到磁盘中，然后标记 redo log 哪些记录可以被擦除，接着对旧的 redo log 记录进行擦除，等擦除完旧记录腾出了空间，checkpoint 就会往后移动(图中顺时针)，然后 MySQL 恢复正常运行，继续执行新的更新操作

### binlog 归档日志

#### 内容

MySQL 在完成一条更新操作后，Server 层会生成一条 binlog，等事务提交时会将该事务执行过程中产生的所有 binlog 统一写入binlog 文件

binlog 文件是以二进制形式记录了所有数据库表结构变更和表数据修改的日志，**不会记录查询类的操作**，比如 SELECT 和 SHOW 操作

binlog 有 3 种格式类型：
    - STATEMENT：每一条修改数据的 SQL 都会被记录到 binlog 中(逻辑日志)，但存在动态函数主从库执行结果不一致的问题
    - ROW：记录行数据最终的修改值，缺点是每行数据的变化结果都会被记录，使 binlog 文件过大
    - MIXED：包含了 STATEMENT 和 ROW 模式，根据不同的情况自动使用 ROW 模式和 STATEMENT 模式

#### 刷盘

事务执行过程中，先把日志写到 binlog cache(Server 层)，事务提交的时候，再把 binlog cache 写到 binlog 文件中，一个事务的 binlog 是不能被拆开的，因此无论这个事务有多大(比如有很多条语句)，也要保证一次性写入

> ![binlogcache.drawio.png (721×461) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/binlogcache.drawio.png)

Mysql中每个线程都有binlog cache，通过参数`binlog_cache_size`控制大小，通过参数`sync_binlog`控制刷盘频率
- sync_binlog = 0(default)：每次提交事务只write不fsync，由操作系统决定刷盘时机
- sync_binlog = N：每次提交事务write，提交N个事务刷盘一次

#### 主从复制

Mysql 的主从复制依赖于binlog
>  ![主从复制过程.drawio.png (991×401) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E8%BF%87%E7%A8%8B.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

Mysql中的主从复制一般是**异步**的，通常分为三个阶段：
- *写入 Binlog*：主库写 binlog 日志，提交事务，并更新本地存储数据
- *同步 Binlog*：把 binlog 复制到所有从库上，每个从库把 binlog 写到relay log中
- *回放 Binlog*：从库读取relay log，回放 binlog，并更新存储引擎中的数据

从库数量过多会导致所需线程数过多，开销较大
实际使用中，一个主库一般跟 2～3 个从库(1 套数据库，1 主 2 从 1 备主)，这就是一主多从的 *MySQL 集群结构*

主从复制类型：
- **同步复制**：MySQL 主库提交事务的线程要等待所有从库的复制成功响应，才返回客户端结果。这种方式在实际项目中，基本上没法用，原因有两个：一是性能很差，因为要复制到所有节点才返回响应；二是可用性也很差，主库和所有从库任何一个数据库出问题，都会影响业务。
- **异步复制**(默认模型)：MySQL 主库提交事务的线程并不会等待 binlog 同步到各从库，就返回客户端结果。这种模式一旦主库宕机，数据就会发生丢失。
- **半同步复制**：MySQL 5.7 版本之后增加的一种复制方式，介于两者之间，事务线程不用等待所有的从库复制成功响应，只要一部分复制成功响应回来就行，比如一主二从的集群，只要数据成功复制到任意一个从库上，主库的事务线程就可以返回给客户端。**这种半同步复制的方式，兼顾了异步复制和同步复制的优点，即使出现主库宕机，至少还有一个从库有最新的数据，不存在数据丢失的风险**

### 差异

undo log 和 redo log 这两个日志都是 Innodb 存储引擎生成的，binlog由Mysql的Sever层生成

redo log VS binlog：
- 生成对象不同：redo log 由InnoDB实现，binlog由Mysql的Sever层实现
- 内容不同：redo log 是物理日志，内容基于数据页，只记录未被刷入磁盘的buffer pool数据日志；binlog为二进制，可以基于sql语句，数据本身或二者混合，保存全量日志
- 写入方式不同：binlog为追加写，在事务提交时写入；redo log为循环写，有多种提交时机
- 作用不同：redo log 用于崩溃恢复，binlog用于备份恢复，主从复制

redo log VS undo log：
- redo log 记录了此次事务*完成后*的数据状态，记录的是更新**之后**的值
- undo log 记录了此次事务*开始前*的数据状态，记录的是更新**之前**的值

## 其他

### 伪列

伪列的行为与表中的列相同，但并未存储具体数值，因此伪列只具备读属性
