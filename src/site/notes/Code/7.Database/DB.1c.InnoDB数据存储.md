---
{"dg-publish":true,"permalink":"/Code/7.Database/DB.1c.InnoDB数据存储/","title":"InnoDB 数据存储","noteIcon":""}
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
> ![COMPACT.drawio.png (2336×562) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/COMPACT.drawio.png)

### 额外信息

*变长字段长度列表*(1~2bytes)只出现在数据表有变长字段的时候，**逆序存放**记录中变长字段的真实数据占用的字节数，**不保存值为 NULL 的变长字段的长度**
对于每个变长字段，如果字段长度小于 256 字节，变长字段长度列表使用1 byte 来表示长度，如果字段长度大于等于 256 bytes，则使用2 bytes来表示长度

*NULL 值列表*(>=1byte)只出现在数据表有字段未定义为 NOT NULL 的时候，用字节的二进制位以 bitmap 形式**逆序标记**当前记录中的 NULL 值

*记录头信息*中包含:
- `delete_mask`：标识数据是否已经被删除
- `next_record`：指向下一条记录的指针
- `record_type`：当前行记录的类型
- ......

>[!Note] 逆序存放
>记录头信息中指向下一个记录的指针指向的是下一条记录中额外信息和真实数据的分界处
>逆序存放可以使得位置靠前的记录的真实数据和数据对应的字段长度信息同时存在于一个 CPU Cache Line 中，提高 CPU Cache 的命中率

### 真实数据

#### 隐藏字段

真实数据中包含三个隐藏字段：
- `row_id`：6bytes，隐式唯一约束，非必需，当表没有主键或唯一约束列时 InnoDB 为记录添加 row_id 隐藏字段
- `trx_id`：6bytes，生成该数据的事务 id
- `roll_pointer`：7bytes，记录上一个版本的指针

> [!Note] varchar(n) 中 n 最大取值
> MySQL 规定除了 TEXT、BLOBs 等大对象类型之外，一行记录中所有字段(不包括隐藏字段和记录头信息)占用的字节长度之和不能超过 65535 个字节
>**所有字段的长度 + 变长字段字节数列表所占用的字节数 + NULL 值列表所占用的字节数 <= 65535**
>因此 varchar(n) 中 n 的最大取值为
> $$\frac{(65535-len(变长字段长度列表)-len(NULL值列表))}{max(单字符字节数)}$$

### 行溢出

当一条记录的大小(<=65535bytes)大于一个页的大小(16kb=16384bytes)时会发生行溢出，使用额外的*溢出页*存储数据

Compact 在记录的真实数据处仅保存一部分数据，然后用20字节指针指向溢出页地址
Compressed 和 Dynamic 采用完全行溢出，真实数据处仅存储20字节指针，完全使用溢出页存放实际数据
此外 Compressed 行格式可以将重复的数据压缩成一个字典，然后只存储字典中的索引和偏移量以大大减少存储空间，但是解压缩过程增加 CPU 开销

## 数据页

InnoDB 以页为单位读取数据

>[从数据页的角度看 B+ 树 | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/mysql/index/page.html)
> ![243b1466779a9e107ae3ef0155604a17.png (692×843) (xiaolincoding.com)](https://cdn.xiaolincoding.com//mysql/other/243b1466779a9e107ae3ef0155604a17.png)

文件头：页信息，存在指向前后数据页的指针，使得页集合构成一个双向链表
页头：数据页状态信息
最大最小记录：两个虚拟的行记录，分别代表页中的**绝对最大索引记录和绝对最小索引记录**
用户记录：存储行记录
空闲空间：还未使用的空间
页目录：存储用户记录的相对位置，起索引作用
文件尾：用于校验页是否完整

数据页中的记录**按照主键顺序**组成单向链表，便于插入删除，通过页目录的索引实现快速访问

数据页将**未标记为已删除的记录按序从小到大**分组存储，分组数量遵循原则：
- 第一个分组仅包含页内虚拟绝对最小记录
- 最后一个分组记录条数为1-8条
- 其余分组记录条数为4-8条
每组的最后一条记录为该组最大记录，该记录头信息中存储该组的记录数量
*页目录*存储每组最后一条记录的地址偏移量，称之为*槽*
通过槽可以二分查找确定记录组，再通过顺序查找确定目标行记录

> ![261011d237bec993821aa198b97ae8ce.png (1080×1228) (xiaolincoding.com)|450](https://cdn.xiaolincoding.com//mysql/other/261011d237bec993821aa198b97ae8ce.png)

### 索引页

索引页也为16K，记录数据页/索引页的最小主键 id 和页号

> ![585429e5078566bda9b2fa18f85215af.png (1080×521) (xiaolincoding.com)](https://cdn.xiaolincoding.com//mysql/other/585429e5078566bda9b2fa18f85215af.png)

## Buffer pool

> [揭开 Buffer Pool 的面纱 | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/mysql/buffer_pool/buffer_pool.html#buffer-pool-%E7%BC%93%E5%AD%98%E4%BB%80%E4%B9%88)
> [Innodb Buffer Pool详解 - 墨天轮 (modb.pro)](https://www.modb.pro/db/607938)
> [详解MySQL中的Buffer Pool，深入底层带你搞懂它！ - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1828772)
> [[玩转MySQL之十]InnoDB Buffer Pool详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/65811829)

InnoDB通过将磁盘中取出的数据缓存至内存的buffer pool中提升查询性能

Buffer pool默认配置为`128MB`，可以通过`innodb_buffer_pool_size` 参数设置 Buffer Pool 的大小，一般建议设置成可用物理内存的60%~80%

在Mysql服务启动时InnoDB为buffer pool申请一片连续的内存空间，按照默认`16kb`的大小划分为缓存页

### 内部结构

> ![](https://oss-emcsprod-public.modb.pro/image/auto/modb_20230128_4065fd32-9eb0-11ed-9afe-38f9d3cd240d.png)

整个  buffer pool 由多个 instance 组成，个数等于 `innodb_buffer_pool_size/innodb_buffer_pool_instances`

Instance 用于并发读取与写入，相互之间没有锁竞争关系

每个 instance 被均匀分为多个 chunk，chunk 包含缓存页和与缓存页对应存在的*控制块*
控制块内含缓存页的表空间、页号、缓存页地址、链表节点等信息

Buffer pool 中缓存页的种类包括*数据页*、*索引页*、*插入缓存页*、*undo 页*、*自适应 hash 页*、*锁信息页*以及 *SDI (结构化字典信息)页*

控制块和缓存页间的灰色部分称为*碎片空间*

> ![freelist.drawio.png (1217×597) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/freelist.drawio.png)

*空闲页*的管理通过**free 链表**实现，其将空闲缓存页的控制块作为双向链表的节点，**头节点为特殊节点**，包含链表的真正头节点地址，尾节点地址，以及当前链表中节点的数量等信息
脏页的管理通过**flush 链表**实现，与 free 链表结构相同，此外还存在 LRU list，Clock list 等链表

### 数据管理-LRU

Buffer pool 使用最近最少使用(LRU)算法的变体将很少使用的数据从缓存中清除

简单的 LRU 为所有数据维护一个顺序链表，不断将新读数据页插入表头并删除表尾数据页，存在两大问题
- **预读失效**：程序具有空间局部性，Mysql 在加载数据页时会预读加载相邻的数据页，简单的 LRU 算法会导致这些未访问的数据页占据链表前端
- **buffer pool 污染**：当某一个 sql 扫描大量数据时会将 buffer pool 里的所有数据都替换出去，导致大量热数据被淘汰

InnoDB 使用的 LRU 变体将 LRU 划分为*young*和*old*两个区域，young 区域在 LRU 链表的前半部分，old 区域在后半部分
新的缓存页最初被加入 old 区域头部，**已在 old 区域停留超过一定时间的缓存页在再次被访问时**被插入 young 区域头部
Old 区域解决预读失效，停留时间限制解决 pool 污染
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

InnoDB 使用 **WAL (Write-Ahead Logging)技术**，通过先写 [[Code/7.Database/DB.1d.Mysql日志#redo log 重做日志\|redo log 日志]]的方式避免宕机造成的 buffer pool 数据丢失，保证持久性
写入redo log 比直接将 Buffer Pool 中修改的数据页写入磁盘(刷脏)要快：
- 刷脏是随机IO，每次修改的数据位置随机，写redo log是追加操作，属于顺序IO
- 刷脏以数据页为单位，对任何小修改都要整页写入，redo log 中只包含真正的操作内容，无效 IO 大大减少

脏页刷新：
- redo log日志满了将所有脏页写入
- buffer pool清除数据时先将数据中的脏页写入
- Mysql判定处于空闲状态时定期刷入适量脏页
- Mysql 正常关闭前将所有脏页写入

可通过 `innodb_max_dirty_pages_pct_lwm` 参数(default:0)启用*脏页预刷新*，在脏页占有率超过 `innodb_max_dirty_pages_pct` (default:75)时强制写入

#### 主从同步

主从缓存同步分为 snapshot、transmit、recover 三个步骤：

snapshot将主库buffer pool状态逻辑导出到主库本地的ib_bp_info文件中，具体过程如下（buf_snapshot）：
1. 逐一扫描每个所有buffer pool instance的LRU list和CLOCK list中的每个页面，将页面的space_id和page_no汇总到以space_id为key，unordered_set为value的lru_maps中。扫描结束后，所有页面按照space_id进行了初步归类
2. 逐一处理每个 space_id 下的页面。通过页面的前后节点指针，将相邻的页面串联成页面链表，并用最左页面、最左页面的第一条用户记录、最右页面、最右页面最后一条用户记录代表该页面链表，存入 multi_ranges 中（link_page）
3. 将 multi_ranges 中所有 record range 按照一定格式落盘到本地 ib_bp_info 文件中。

针对如何传输ib_bp_info文件，有很多备选方案：采用scp来传输文件误操作的可能性比较高，并且经常修改传输脚本；采用binlog传输会加重binlog的负担，加大主从延迟的风险，方案复杂，风险大；采用内建表的方式同步文件又担心引入兼容性的问题。主从缓存通过最终选择了transmit方案，即在slave新创建一种与IO线程相似，需要与master建立tcp连接的transmit线程（handle_slave_transmit），该线程负责向master发送ib_bp_info文件传输申请，并负责接收ib_bp_info文件。transmit方案不侵入主从复制的原有逻辑，对于不支持transmit的主库，也仅仅是无法从主库获取ib_bp_info文件，不存在兼容问题。

recover 将 ib_bp_info 文件中代表的数据加载到从库 buffer pool 中。Recover 可以在跨机迁移前、版本升级前、主从切换前或切换过程中进行，也可以定期将主库的 buffer pool 同步到从库，随时做好切换准备。Recover 具体过程如下（buf_recover）：
1. 按固定格式识别ib_bp_info中的record_range信息
2. 在从库通过B+树搜索record_range最左页面的第一条用户记录和最右页面的最后一条用户记录对应的两个页面
3. 将上述两个页面之间的所有页面加载到 buffer pool 中。为了避免新加载的页面互相淘汰，recover 操作还支持将新页面直接加载到 LRU list 的头部而不是 midpoint
4. 重复1-3步骤，直至ib_bp_info中所有recored_range被处理完成

## Change buffer

一条普通的DML可能因为需要修改多个二级索引，而带来大量随机IO

Innodb 存储引擎引入 change buffer 解决二级索引随机 IO 问题 

当待插入、删除、修改的记录所在的二级索引页面不在 buffer pool 中时，修改内容会以记录的形式缓存到 change buffer 中
当二级索引页面最终被读入 buffer pool 中时，检查 change buffer 中是否有该页面的修改记录，如果存在需要将修改记录合并到新读入的二级索引页面上，再返回

Change buffer 新写入的内容也会写入到 redo log 从而保证 change buffer 持久性
