---
{"dg-publish":true,"permalink":"/Code/6.Database/DB.1c.InnoDB数据管理/","title":"InnoDB 数据存储","noteIcon":""}
---


# InnoDB 数据存储

```bash
var/lib/mysql/$database_name  数据库默认地址
db.opt  存储当前数据库的默认字符集和字符校验规则
$table_name.frm  表结构
$table_name.ibd  表数据
```
表数据既可以放在共享表空间文件(ibdata1)里，也可以存放在独占表空间文件($table_name.ibd)，默认为后者

表空间由**段(segment)**、**区(extent)**、**页(page)**、**行(row)** 组成
>![表空间结构.drawio.png (686×651) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/%E8%A1%A8%E7%A9%BA%E9%97%B4%E7%BB%93%E6%9E%84.drawio.png)

行：数据库表中记录的存放方式，每行记录根据不同的行格式有不同的存储结构

页：**InnoDB 读取数据的基本单位**，读取记录时以页为单位整体读入内存，**每页的默认大小为16KB**，页在磁盘中不一定是连续的

区：表中数据量大时，以区代替页为单位为索引分配空间，使链表中相邻的页在磁盘中的物理位置相邻，便于顺序读取，**每个区的大小为1MB，也即64个连续页**

段：一般分为数据段，索引段，回滚段等
- 索引段：存放**B+树非叶子节点**的区的集合
- 数据段：存放**B+树叶子节点**的区的集合
- 回滚段：存放**回滚数据**的区的集合

## 行记录

InnoDB提供四种行格式，分别为Redundant、Compact、Dynamic和 Compressed
Redundant 为 V5.0前使用的非紧凑行格式，V5.1后默认使用紧凑行格式 Compact，V5.7后默认使用改进紧凑行格式 Dynamic

Compressed 和 Dynamic 行格式与 Compact 的主要区别在于处理行溢出数据的方式不同

Compact 行格式：
> ![COMPACT.drawio.png (2336×562) (xiaolincoding.com)|900](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/COMPACT.drawio.png)

### 额外信息

*变长字段长度列表*(1~2 byte)只出现在数据表有变长字段的时候，**逆序存放**记录中变长字段的真实数据占用的字节数，**不保存值为 NULL 的变长字段的长度**
对于每个变长字段，如果字段长度小于 **256 byte**，变长字段长度列表使用1 byte 来表示长度，如果字段长度大于等于 256 byte，则使用2 byte 来表示长度

*NULL 值列表*(>=1 byte)只出现在数据表有字段未定义为 NOT NULL 的时候，用字节的二进制位以 bitmap 形式**逆序标记**当前记录中的 NULL 值

*记录头信息*中包含:
- `delete_mask`：标识数据是否已经被删除
- `next_record`：指向下一条记录中额外信息和真实数据分界处的指针
- `record_type`：当前行记录的类型
- ……

>[!Note] 逆序存放
>记录头信息中指向下一个记录的指针指向的是下一条记录中额外信息和真实数据的分界处
>逆序存放可以使得位置靠前的记录的真实数据和数据对应的字段长度信息同时存在于一个 CPU Cache Line 中，提高 CPU Cache 的命中率

### 真实数据

#### 隐藏字段

真实数据中包含三个隐藏字段：
- `row_id`：6 byte，隐式唯一约束，**非必需**，当表没有主键或唯一约束列时 InnoDB 为记录添加 row_id 隐藏字段
- `trx_id`：6 byte，生成该数据的事务 id
- `roll_pointer`：7 byte，记录上一个版本的指针

> [!Note] varchar(n) 中 n 最大取值
> MySQL 规定除了 TEXT、BLOBs 等大对象类型之外，一行记录中所有字段(不包括隐藏字段和记录头信息)占用的字节长度之和不能超过 65535 个字节
>**所有字段的长度 + 变长字段字节数列表所占用的字节数 + NULL 值列表所占用的字节数 <= 65535**
>因此 varchar(n) 中 n 的最大取值为
> $$\frac{(65535-len(变长字段长度列表)-len(NULL值列表))}{max(单字符字节数)}$$

### 行溢出

当一条记录的大小(<= 65535 byte)大于一个页的大小(16 kb=16384 byte)时会发生行溢出，**使用额外的*溢出页*存储数据**

Compact 在记录的真实数据处仅保存一部分数据，然后用20字节指针指向溢出页地址
Compressed 和 Dynamic 采用完全行溢出，真实数据处仅存储20字节指针，完全使用溢出页存放实际数据
此外 Compressed 行格式可以将重复的数据压缩成一个字典，然后只存储字典中的索引和偏移量以大大减少存储空间，但是解压缩过程增加 CPU 开销

## 页结构

InnoDB 以页作为内存和磁盘交互的基本单位读取数据，每个页的默认大小为`16KB`

> [从数据页的角度看 B+ 树 | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/mysql/index/page.html)

### B+树


<div class="transclusion internal-embed is-loaded"><a class="markdown-embed-link" href="/code/6-database/db-1a-mysql/#b" aria-label="Open link"><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="svg-icon lucide-link"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"></path><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"></path></svg></a><div class="markdown-embed">



### B+树索引

> [为什么 MySQL 采用 B+ 树作为索引？ | 小林coding ](https://xiaolincoding.com/mysql/index/why_index_chose_bpuls_tree.html#%E4%BB%80%E4%B9%88%E6%98%AF-b-%E6%A0%91)

> ![7c635d682bd3cdc421bb9eea33a5a413.png (1080×509) (xiaolincoding.com)](https://cdn.xiaolincoding.com//mysql/other/7c635d682bd3cdc421bb9eea33a5a413.png)

特点：基于磁盘的多叉平衡树，通常只有3~4层，磁盘I/O次数少，**查询效率高**

B+树是一种多叉树，**叶子节点存放实际数据(索引+记录)**，非叶子节点只存放索引
- 每个节点里的数据**按主键顺序存放**
- 每一层父节点的索引值都会出现在下层子节点的索引值中，并且是子节点中所有索引的最大或最小
- 每个叶子节点都包含有前后叶子节点的指针，**构成双向链表**，有助于范围查找
- B+ 树在删除根节点的时候，由于存在冗余的节点，所以不会发生复杂的树的变形

所有 B+树都从高度为1的树开始慢慢增加高度
B+ 树在删除根节点的时候，由于存在冗余的节点，所以不会发生复杂的树的变形

InnoDB 里的 B+ 树中的**每个节点都是一个数据页**，**非叶子节点的大小受页限制**

#### 插入删除

> [B树和B+树的插入、删除图文详解 - nullzx - 博客园 (cnblogs.com)](https://www.cnblogs.com/nullzx/p/8729425.html)

记叶子节点最多存储数据为 m 条

B+树插入
1. 若为空树，创建一个叶子结点，然后将记录插入其中，此时这个叶子结点也是根结点，插入操作结束
2. 针对叶子类型结点
    1. 根据 key 值找到叶子结点，向这个叶子结点插入记录
    2. 插入后，若当前结点 key 的个数小于等于 m-1，则插入结束
    3. 否则将这个叶子结点分裂成左右两个叶子结点，左叶子结点包含前 m/2 个记录，右结点包含剩下的记录，将第 m/2+1 个记录的 key 进位到父结点(索引类型结点)中，进位到父结点的 key 左孩子指针向左结点,右孩子指针向右结点。将当前结点的指针指向父结点
3. 针对索引类型结点
    1. 若当前结点 key 的个数小于等于 m-1，则插入结束
    2. 否则，将这个索引类型结点分裂成两个索引结点，左索引结点包含前(m-1)/2 个 key，右结点包含 m-(m-1)/2 个 key，将第 m/2 个 key 进位到父结点中，进位到父结点的 key 左孩子指向左结点, 进位到父结点的 key 右孩子指向右结点，将当前结点的指针指向父结点，继续对父节点进行循环

B+树删除
1. 删除叶子结点中对应的 key，删除后若结点的 key 的个数大于等于 `Math.ceil(m-1)/2 – 1`，删除操作结束，否则继续
2. 若兄弟结点 key 有富余(大于 `Math.ceil(m-1)/2 – 1`)，向兄弟结点借一个记录，同时用借到的 key 替换当前结点和兄弟结点共同的父结点中的 key，删除结束
3. 若兄弟结点中没有富余的 key，则当前结点和兄弟结点合并成一个新的叶子结点，并删除父结点中的一个对应 key，将当前结点指向父结点(索引结点)
4. 若索引结点的 key 的个数大于等于 `Math.ceil(m-1)/2 – 1`，则删除操作结束
    1. 若兄弟结点有富余，父结点 key 下移，兄弟结点 key 上移，删除结束
    2. 否则当前结点，兄弟结点及父结点下移 key 合并成一个新的结点
5. 将当前结点指向父结点，继续对父节点进行循环

注意，**通过 B+树的删除操作后，索引结点中存在的 key，不一定在叶子结点中存在对应的记录**

#### 比较其他树

- 二叉查找树：极端情况下查找数据的时间复杂度为 O(n)
- 自平衡二叉树：树的高度较高，需要多次磁盘 IO，每次插入新节点的复杂度不定
- 红黑树：保证每次插入最多只需要三次旋转就能达到平衡，树的高度较高
- B 树：在数据查询中比平衡二叉树效率要高，但读取过程中非叶子节点中的数据也被加载进缓存，非叶子节点内可存储索引少


</div></div>


### 数据页 - 叶子节点

> ![|475](https://cdn.xiaolincoding.com//mysql/other/243b1466779a9e107ae3ef0155604a17.png)

- *文件头*：页信息，存在指向前后数据页的指针，使得**页集合构成一个双向链表**
- *页头*：数据页状态信息
- *最大最小记录*：两个虚拟的行记录，分别代表页中的**绝对最大索引记录和绝对最小索引记录**
- *用户记录*：存储行记录
- *空闲空间*：还未使用的空间
- *页目录*：存储用户记录的相对位置，起索引作用
- *文件尾*：用于校验页是否完整

数据页中的记录**按照主键顺序**组成单向链表，便于插入删除，通过页目录的索引实现快速访问

数据页将**未标记为已删除的记录按序从小到大**分组存储，分组数量遵循原则：
- 第一个分组仅包含页内绝对最小记录(虚拟记录)
- 最后一个分组记录条数为1-8条
- 其余分组记录条数为4-8条
每组的最后一条记录为该组最大记录(虚拟记录)，该记录头信息中存储该组的记录数量
*页目录*存储每组最后一条记录的地址偏移量，称之为*槽*
通过槽可以**二分查找**确定记录组，再通过顺序查找确定目标行记录

> ![261011d237bec993821aa198b97ae8ce.png (1080×1228) (xiaolincoding.com)|525](https://cdn.xiaolincoding.com//mysql/other/261011d237bec993821aa198b97ae8ce.png)

### 索引页 - 非叶子节点

索引页也为16K，记录数据页/索引页的最小主键 id 和页号，记录**按照索引键顺序**组成单向链表

> ![585429e5078566bda9b2fa18f85215af.png (1080×521) (xiaolincoding.com)|725](https://cdn.xiaolincoding.com//mysql/other/585429e5078566bda9b2fa18f85215af.png)

## Buffer pool

> [揭开 Buffer Pool 的面纱 | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/mysql/buffer_pool/buffer_pool.html#buffer-pool-%E7%BC%93%E5%AD%98%E4%BB%80%E4%B9%88)
> [Innodb Buffer Pool详解 - 墨天轮 (modb.pro)](https://www.modb.pro/db/607938)
> [详解MySQL中的Buffer Pool，深入底层带你搞懂它！ - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1828772)
> [[玩转 MySQL 之十]InnoDB Buffer Pool 详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/65811829)
> [(十二)MySQL之内存篇：深入探寻数据库内存与Buffer Pool的奥妙！ - 掘金 (juejin.cn)](https://juejin.cn/post/7159071309354254373)

InnoDB 通过将磁盘中取出的数据缓存至内存的 buffer pool 中提升查询性能

Buffer pool默认配置为`128MB`，可以通过`innodb_buffer_pool_size` 参数设置 Buffer Pool 的大小，一般建议设置成可用物理内存的60%~80%

在 Mysql 服务启动时 InnoDB 为 buffer pool 申请一片连续的内存空间，按照与磁盘数据相同的默认 `16kb` 的大小划分为缓存页

> ![](https://oss-emcsprod-public.modb.pro/image/auto/modb_20230128_4065fd32-9eb0-11ed-9afe-38f9d3cd240d.png)

整个  buffer pool 由多个 instance 组成，instance 用于并发读取与写入，相互之间没有锁竞争关系

每个 instance 被均匀分为多个 chunk，包含缓存页和对应存在的*控制块*
控制块内含缓存页的表空间、页号、缓存页地址、链表节点等信息
缓存页的种类包括*数据页*、*索引页*、*插入缓存页*、*undo 页*、*自适应 hash 页*、*锁信息页*以及 *SDI (结构化字典信息)页*
控制块和缓存页间的部分为*碎片空间*

> ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/85dcd0e4026d403fbabadc193d4b2bfc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 缓存页结构

#### Change buffer 写入缓冲

一条普通的 DML 语句可能因为需要修改多个二级索引，而带来大量随机 IO
InnoDB 存储引擎引入 change buffer 解决二级索引随机 IO 问题

当待插入、删除、修改的记录所在的二级索引页面不在 buffer pool 中时，修改内容会以记录的形式缓存到 change buffer 中
当二级索引页面最终被读入 buffer pool 中时，检查 change buffer 中是否有该页面的修改记录，如果存在需要将修改记录合并到新读入的二级索引页面上，再返回

change buffer 新写入的内容也会写入到 redo log，从而保证 change buffer 持久性

当一条写入语句执行时，流程如下：
1. 判断要变更的数据页是否被载入到内存
2. 如果内存中有对应的数据页，则直接变更缓冲区中的数据页，完成标记后则直接返回
3. 如果内存中没有对应的数据页，则将要变更的数据放入到 change buffer 中，然后返回

不是所有的写入动作都可以在内存中完成， **插入的数据字段不能具备唯一约束或唯一索引**，因为存在唯一字段的表在插入数据前必须要先依赖磁盘中的表数据判断表中是否存在相同值
表的主键为一个自增 ID 时，由于 Mysql 维护的ID 会由 `MySQL` 来生成，绝对不会出现重复值的，对于这种情况，会将要插入的数据放到写入缓冲中

对于具备唯一约束或唯一索引的表，插入数据时对次级非唯一索引树的维护会用到 change buffer

**写入磁盘时机：**
-   当一条 SQL 需要用到对应的索引键查询数据时，会触发后台线程执行刷盘工作 
-   当 change buffer 内存空间不足时，会触发后台线程执行刷盘工作
-   当距离上一次刷盘的时间，间隔达到一定程度(默认10s)，会触发后台线程执行刷盘工作
-   MySQL 关闭时也会触发后台线程执行刷盘工作

#### 索引页

`Buffer Pool` 中有一块专门的区域：`Index Page`，专门用来存放载入的索引数据，存储这些数据的缓冲页，则被称之为索引页
Mysql 启动时会将当前库中所有已存在的索引的根节点页放入到内存缓冲区中，对于需要走索引查询的 sql 语句直接以相应的索引根节点为起始，然后去走索引查找数据，避免了全盘查找索引根节点的操作
随着运行时间的增长，也会将一些非根节点的索引页载入内存中

#### 锁空间

锁空间是专门用来存储锁结构的一块内存区域
同时还会存储一些并发事务的链表，例如死锁检测时需要的事务等待链表、锁的信息链表等
当锁空间内存不足时，就会导致行锁粗化成表锁，以此来减少锁结构的数量，释放一定程度上的内存，但此时并发冲突就会变高

#### 数据字典

`InnoDB` 引擎中主要存在 `SYS_TABLES、SYS_COLUMNS、SYS_INDEXES、SYS_FIELDS` 这四张系统表，也被称为`InnoDB`的内部表，主要是用来维护用户定义的所有表的各种信息
这四张表在载入内存前位于 `.ibdata` 文件中，在 `MySQL` 启动时会开始加载，载入内存后就会放入到 `Dict Info` 这块区域，当利用 `show` 语句查询表的结构信息时，就会在字典信息中检索数据

#### 日志缓冲区

Buffer Pool 主要存在两个日志缓冲区，即 `undo_log_buffer、redo_log_buffer`，分别对应着撤销日志和重做日志，作用主要是用来提升日志记录的写入速度，执行 `SQL` 时先写缓冲区，再由后台线程去刷写日志

#### 自适应哈希索引

在 `MySQL` 运行过程中，`InnoDB` 引擎会对表上的索引做监控，如果某些数据经常走索引查询，那 `InnoDB` 就会为其建立一个哈希索引，以此来提升数据检索的效率，并且减少走 `B+Tree` 带来的开销，由于这种哈希索引是运行过程中，`InnoDB` 根据 `B+Tree` 的索引查询次数来建立的，因此被称之为自适应哈希索引

自适应哈希索引和普通哈希索引的区别在于，普通哈希索引是在创建索引时将结构声明为 `Hash` 结构，这种索引会以索引字段的整表数据建立哈希，而自适应哈希索引是根据缓冲池的 `B+` 树构造而来，只会基于热点数据构建，因此建立的速度会非常快，毕竟无需对整表都建立哈希索引

对于自适应哈希索引的使用情况，可以通过`show engine innodb status \G;`命令查看，但哈希索引由于自身特性的原因，因此也仅只能用于等值查询的场景，无法支持排序、范围查询

在`MySQL8.0`以下的版本中，如果同时删除一张大表的很多数据，有可能会因为自适应哈希索引的原因，造成线上`MySQL`出现抖动，不过该问题在`MySQL8.x`版本中已经被修复，但如若你的`MySQL`版本在此之下，那尽量不要在业务高峰期删除大量数据。

### 数据管理

> ![freelist.drawio.png (1217×597) (xiaolincoding.com)|675](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/freelist.drawio.png)

**空闲页的管理通过 free 链表实现**，其将空闲缓存页的控制块作为双向链表的节点，**头节点为特殊节点**，包含链表的真正头节点地址，尾节点地址，以及当前链表中节点的数量等信息

当线程变更了内存中的数据页之后，会先对这个数据页做个标记，然后直接给客户端返回执行成功的响应，在这个过程中被线程变更并标记过的数据页被称之为标记页/脏页
**脏页的管理通过 flush 链表实现**，与 free 链表结构相同，此外还存在 LRU list，Clock list 等链表

#### 内存淘汰 - LRU

空闲页和脏页不会纳入淘汰范围，对脏页的淘汰会导致未落盘数据的丢失，因此 `LRU` 链表是由已使用、但未曾变更过的缓冲页组成的

因为所有的链表都是由控制块作为节点构建的，而一个控制块中只有一根指针，也意味着一个控制块同时只能加入一个链表中，所以不可能出现一个缓冲页既处于`LRU`链表，又位于`Flush`链表中
有些数据页会在 Flush 和 LRU 两个链表之间 “跳动”：
-   当 `LRU` 链表中的一个数据页发生变更后，会从 `LRU` 链表转到 `Flush` 链表 
-   当标记页中的变更数据落盘后，此时标记页又会从 `Flush` 链表回到 `LRU` 链表

Buffer pool 使用**最近最少使用(LRU)算法的变体**将很少使用的数据从缓存中清除

简单的 LRU 为所有数据维护一个顺序链表，不断将新读数据页插入表头并删除表尾数据页，存在两大问题
- **预读失效**：程序具有空间局部性，Mysql 在加载数据页时会预读加载相邻的数据页，简单的 LRU 算法会导致这些未访问的数据页占据链表前端
- **buffer pool 污染**：当某一个 sql 扫描大量数据时会将 buffer pool 里的所有数据都替换出去，导致大量热数据被淘汰

InnoDB 使用 Old 区域解决预读失效，使用晋升限制解决 pool 污染
InnoDB 使用的 LRU 变体将 LRU 划分为*young*和*old*两个区域，young 区域在 LRU 链表的前半部分，old 区域在后半部分
新的缓存页最初加入 old 区域头部，**已在 old 区域停留超过一定时间(默认 1s)的缓存页在再次被访问时**插入 young 区域头部
> ![young+old.png (842×212) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/young%2Bold.png)

old 区域占整体的比例通过 `innodb_old_blocks_pct` 参数设置，默认是 37，也即3/8
old 区域时间限制通过 `innodb_old_blocks_time` 参数设置，默认是 1000 ms

另外 MySQL 针对 young 区域进行了优化，为了防止 young 区域节点频繁移动到头部，只有在 young 区域中位于后面的 3/4的节点被访问时才会移动到头部

### 具体操作

- 加速读：读取数据时，如果数据页存在于 buffer pool 中，客户端直接读取池中数据
- 加速写：修改数据时，首先修改 buffer pool 中数据所在页，然后将该页设置为*脏页*，通过后台进程将脏页写入磁盘

#### 预热

重启 MySQL 后，空 Buffer Pool 需要一定时间才能存储有业务频繁使用的数据，该过程称为*预热*
在预热过程中 MySQL 数据库性能较差，为了减短预热过程，可以在 MySQL 关闭前，把 Buffer Pool 中的页面信息保存到磁盘，等到 MySQL 启动时，再根据之前保存的信息把磁盘中的数据加载到 Buffer Pool 中

V5.6后 InnoDB 支持*单机预热*功能
- Dump - 将重启前 buffer pool 的 LRU list 中的页面信息记录至本地  `innodb_buffer_pool_filename` 文件中
- Load - 重启后将本地文件中的页面加载至 Buffer Pool 中

#### 预读

预读请求是一个 IO 请求，异步地在缓冲池中预先读取多个页面至 Buffer Pool

*线性预读*：一个区域中被顺序读取的页超过或者等于 `innodb_read_ahead_threshold` 变量时，InnoDB 异步地将下一个区域读取至 Buffer Pool 中
*随机预读*：在 V5.5废弃

#### 写入磁盘

InnoDB 将脏页按照第一次修改时间(变脏时间)进行排序，通过 flush 链表进行管理，由后台刷新线程依次刷新到磁盘

InnoDB 使用 **WAL (Write-Ahead Logging)技术**，通过先写 [[Code/6.Database/DB.1d.Mysql日志#redo log 重做日志\|redo log 日志]]的方式避免宕机造成的 buffer pool 数据丢失，保证持久性
写入redo log 比直接将 Buffer Pool 中修改的数据页写入磁盘(刷脏)要快：
- 刷脏是随机IO，每次修改的数据位置随机，写redo log是追加操作，属于顺序IO
- 刷脏以数据页为单位，对任何小修改都要整页写入，redo log 中只包含真正的操作内容，无效 IO 大大减少

脏页刷新：
- redo log日志满了将所有脏页写入
- buffer pool清除数据时先将数据中的脏页写入
- Mysql判定处于空闲状态时定期刷入适量脏页
- Mysql 正常关闭前将所有脏页写入

可通过 `innodb_max_dirty_pages_pct_lwm` 参数(default:0)启用*脏页预刷新*，在脏页占有率超过 `innodb_max_dirty_pages_pct` (default:75)时强制写入

> [!note] WAL 预写日志
> 在计算机科学中，「预写式日志」（Write-ahead logging，缩写 WAL）是关系数据库系统中用于提供原子性和持久性的一系列技术
> 在使用 WAL 的系统中，所有的修改在提交之前都要先写入 log 文件中

#### 主从同步

主从缓存同步分为 snapshot、transmit、recover 三个步骤：

Snapshot 将主库 buffer pool 状态逻辑导出到主库本地的 ib_bp_info 文件中，具体过程如下(buf_snapshot)：
1. 逐一扫描每个所有buffer pool instance的LRU list和CLOCK list中的每个页面，将页面的space_id和page_no汇总到以space_id为key，unordered_set为value的lru_maps中。扫描结束后，所有页面按照space_id进行了初步归类
2. 逐一处理每个 space_id 下的页面，通过页面的前后节点指针，将相邻的页面串联成页面链表，并用最左页面、最左页面的第一条用户记录、最右页面、最右页面最后一条用户记录代表该页面链表，存入 multi_ranges 中（link_page）
3. 将 multi_ranges 中所有 record range 按照一定格式落盘到本地 ib_bp_info 文件中。

针对如何传输ib_bp_info文件，有很多备选方案：采用scp来传输文件误操作的可能性比较高，并且经常修改传输脚本；采用binlog传输会加重binlog的负担，加大主从延迟的风险，方案复杂，风险大；采用内建表的方式同步文件又担心引入兼容性的问题。主从缓存通过最终选择了transmit方案，即在slave新创建一种与IO线程相似，需要与master建立tcp连接的transmit线程（handle_slave_transmit），该线程负责向master发送ib_bp_info文件传输申请，并负责接收ib_bp_info文件。transmit方案不侵入主从复制的原有逻辑，对于不支持transmit的主库，也仅仅是无法从主库获取ib_bp_info文件，不存在兼容问题。

recover 将 ib_bp_info 文件中代表的数据加载到从库 buffer pool 中。Recover 可以在跨机迁移前、版本升级前、主从切换前或切换过程中进行，也可以定期将主库的 buffer pool 同步到从库，随时做好切换准备。Recover 具体过程如下（buf_recover）：
1. 按固定格式识别ib_bp_info中的record_range信息
2. 在从库通过B+树搜索record_range最左页面的第一条用户记录和最右页面的最后一条用户记录对应的两个页面
3. 将上述两个页面之间的所有页面加载到 buffer pool 中。为了避免新加载的页面互相淘汰，recover 操作还支持将新页面直接加载到 LRU list 的头部而不是 midpoint
4. 重复1-3步骤，直至 ib_bp_info 中所有 recored_range 被处理完成
