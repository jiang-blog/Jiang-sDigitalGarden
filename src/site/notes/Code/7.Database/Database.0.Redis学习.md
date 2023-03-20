---
{"dg-publish":true,"permalink":"/Code/7.Database/Database.0.Redis学习/","title":"Redis 学习","noteIcon":""}
---


# Redis 学习

Redis是一个是一个开源的使用 ANSI C 语言编写的高性能的 key-value 数据库

## 基础数据结构

[Redis 命令 | 菜鸟教程 (runoob.com)](https://www.runoob.com/redis/redis-commands.html)

### redisObject

Redis中数据基本由redisObject表示，redisObject中有一个`type`字段以表示数据类型，还有一个`refcount`表示引用数量，`ptr`字段指向数据的实际结构，即使是同一类型，根据使用的编码方式不同，实际结构也可能有所不同。

```C
// Redis 7.0.9

struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;          // 引用计数
    void *ptr;
};
```

> Redis对象在Redis内部被广泛使用，但是为了避免间接访问的开销，最近在许多地方我们只是使用普通的动态字符串，而不是包装在Redis对象中。

#### 整数池

Redis在初始化服务器时会创建数值为0~9999的共10000个整数String对象，并且令它们的`refcount`字段始终为`INT_MAX`

### String

| 操作 | 命令      |
| ---- | --------- |
| 创建 | SET SETNX |
| 查询 | GET MGET  |
| 更新 | SET INCR      |
| 删除 | DEL       |

string对象有三种编码方式：
- INT编码：存一个整型，范围相当于`long`；
- EMBSTR编码：字符串长度小于等于阈值字节，使用EMBSTR编码；
- RAW编码：长度大于阈值字节，则用RAW编码。

阈值字节3.2版本以前为39bytes，3.2版本以后为44bytes
>Redis在Linux上使用jemalloc内存分配器，jemalloc以64字节为阈值区分大小字符串
>redisObject占用的内存大小由redisObject和sdshdr两部分组成redisObject16字节，sdshdr中`len`、`alloc`、`flag`三个字段固定占3个字节，'\\0'占了一个字节，能存放的数据就是64-(16+4)=44

EMBSTR和RAW都由SDS结构组成
EMBSTR下redisObject和SDS是连续的内存
RAW编码下redisObject和SDS的内存分开，由redisObject中数据指针访问SDS

EMBSTR优点是redisObject和SDS两个结构可以一次性分配空间，缺点在于如果重新分配空间，整体都需要再分配，所以**EMBSTR设计为只读**
任何修改操作之后EMBSTR都会变成RAW，理念是发生过修改的字符串通常会认为是易变的

#### SDS

SDS分为sdshdr8、sdshdr16、sdshdr32、sdshdr64，都包含已使用长度`len`以及容量`alloc`
```C
struct __attribute__ ((__packed__)) sdshdr8 {
  uint8_t len; /* used */
  uint8_t alloc; /* excluding the header and null terminator */
  unsigned char flags; /* 3 lsb of type, 5 unused bits */
  char buf[];
};
```

**预留空间大小**：
`len`小于1M的情况下，`alloc`=`len`
`len`大于1M的情况下，`alloc`=1M+`len`

**SDS优点**：
- SDS包含已使用容量字段，O(1)时间快速返回有字符串长度，相比之下，C原生字符串需要O(N)。
- 有预留空间，在广容时如果预留空间足够，就不用再重新分配内存，节约性能，缩容时也可以将减少的空间先保留下来，后续可以再使用。
- 不再以'\\0'作为判断标准，二进制安全，可以很方便地存储一些二进制数据。

### List

List是一组连接起来的String集合，可双端操作

| 操作 | 命令                  |
| ---- | --------------------- |
| 创建 | LPUSH RPUSH           |
| 查询 | LLEN LRANGE           |
| 更新 | LPUSH RPUSH LPOP RPOP |
| 删除 | DEL                   |

V3.2以前List 对象有两种编码方式：
- ZIPLIST - 数组
  - List中所有string对象长度都小于64bytes
  - List中元素个数小于512个
- LINKEDLIST - 链表

V3.2以后List对象采用**QUICKLIST**进行编码
QUICKLIST是以ZIPLIST(Old)/LISTPACK(New)作为单个节点的链表

#### ZIPLIST

ZIPLIST结构为`<zlbytes> <zltail> <zllen> <entry> <entry> … <entry> <zlend>`
- `zlbytes` - ZIPLIST字节总数
- `zltail` - 最后一个节点的相对偏移字节，用于快速访问最后节点
- `zllen` - 元素总数，真实值大于65535时值为0
- `entry` - 节点，结构为 `<prevlen> <encoding> <entry-data>`
  - `prevlen` - 上一节点数据长度，用于向前访问上一节点
  - `encoding` - 编码类型
  - `entry-data` - 实际数据
- `zlend` - 一个特殊的entry节点，表示ZIPLIST的结束

前一节点长度小于255时，`prevlen`占用1byte，否则占用5bytes，255被特殊节点`zlend`使用
`encoding`使用二进制编码保存内容数据的类型和长度

#### LISTPACK

ZIPLIST更新数据时由于`prevlen`可能膨胀，会引起O(N^2)的连锁更新

LISTPACK结构为`<tot-bytes> <num-elements> <element-1> … <element-N> <listpack-end-byte>`
- `tot-bytes` - LISTPACK字节总数
- `num-elements` - 元素总数，真实值大于65535时值为65535
- `element` - 节点
  - `<encoding-type>` - 编码类型
  - `<element-data>` - 实际数据
  - `<element-tot-len>` - 不定长，每个字节首位用于标识是否结束，0为是，剩余bits标识节点长度
- `listpack-end-byte` - 标记LISTPACK结束的值为255的单字节

### Set

Set是一个不重复、无序的String集合

| 操作 | 命令                                               |
| ---- | -------------------------------------------------- |
| 创建 | SADD                                               |
| 查询 | SISMEMBER SCARD SMEMBERS SSCAN SINTER SUNION SDIFF |
| 更新 | SADD SREM                                          |
| 删除 | DEL                                                |

Set编码方式：
- INTSET
  - 元素都为整数
  - 元素数量不超过512个
- HASHTABLE

INTSET结构包括`encoding`，`length`，`contents`
数据通过有序数组存储，查询通过**二分查找**的方式

HASHTABLE结构的Set是值都为空的hash表

#### HASHTABLE

redis 中的 hash table结构如下
```C
// Redis 7.0.9
// redis/deps/hiredis/dict.h
typedef struct dict {
    dictEntry **table;
    dictType *type;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
    void *privdata;
} dict;
```

dict中包含有两个hash table `ht[0]`，`ht[1]`以便于扩容
```C
// Redis 7.0.9
// redis/src/dict.h
struct dict {
    dictType *type;

    dictEntry **ht_table[2];
    unsigned long ht_used[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    /* Keep small vars at end for optimal (minimal) struct padding */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
    signed char ht_size_exp[2]; /* exponent of size. (size = 1<<exp) */
    void *metadata[];           /* An arbitrary number of bytes (starting at a
                                 * pointer-aligned address) of size as defined
                                 * by dictType's dictEntryBytes. */
};

```

redis利用`used/size`作为负载因子来控制hash table的扩容和缩容
周期函数发现负载因子不在阈值范围内则触发操作
- 扩容：
  - 服务器目前没有在执丸行BGSAVE或者BGREWRITEAOF，并且负载因子>=1
  - 服务器目前正在执行BGSAVE或者BGREWRITEAOF，并且负载因子>=5
- 缩容：
  - 当哈希表的负载因子小于0.1时，程序会自动开始对哈希表进行收缩操作。

扩容步骤：
1. 将`h[1]`初始化为新hash table，大小为2倍`used`向2次方幂取整，`rehashidx`置0，开始rehash
2. 迁移`h[0]`数据至`h[1]`，每次增删查改迁移一个数据，周期函数定时迁移一部分数据，更新`rehashidx`
3. `h[1]`和`h[0]`指针对象交换

### Hash

Redis哈希是k-v都为String的哈希表。
| 操作 | 命令         |
| ---- | ------------ |
| 创建 | HSET HSETNX  |
| 查询 | HGETALL HGET HLEN HSCAN |
| 更新 | HSET HSETNX HDEL    |
| 删除 | DEL          |

Hash编码方式：
- ZIPLIST(Old) ->LISTPACK(New)
  - 所有元素的值和键都小于64bytes
  - 元素数量不超过512个
- HASHTABLE

### Sorted Set/ZSet

ZSet是一组按关联分数有序排列的String集合，分数是一个用于排序的抽象概念

| 操作 | 命令                                               |
| ---- | -------------------------------------------------- |
| 创建 | ZADD                                               |
| 查询 | ZCARD ZRANGE ZREVRANGE ZCOUNT ZRANK ZSCORE |
| 更新 | ZADD ZREM                                          |
| 删除 | DEL                                                |

ZSet编码方式：
- ZIPLIST(Old) ->LISTPACK(New)
  - 所有元素都小于64bytes
  - 元素数量不超过128个
- SKIPLIST + HASHTABLE
  - HASHTABLE的值为元素分数

#### SKIPLIST

SKIPLIST是一种可快速查找的多级链表结构

跳表结构
```C
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;  //跳表层数
} zskiplist;
```

跳表节点结构：
```C
// Redis 7.0.9
// redis/src/server.h
/* ZSETs use a specialized version of Skiplists */
typedef struct zskiplistNode {
    sds ele;                            //存储数据
    double score;                       //节点分数
    struct zskiplistNode *backward;     //指向上一个节点的回退指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;  //该层下一个能跳到的节点
        unsigned long span;             //距离下一节点的步数
    } level[];
} zskiplistNode;
```

Redis中跳表的最大层数限制为32层，跳表在插入新节点前选择一个随机的层高，在其`level`数组中存储所有高层节点

### Stream

Stream可以看作一个只能追加的日志结构，**通过流ID唯一标识**

| 操作 | 命令        |
| ---- | ----------- |
| 创建 | XADD        |
| 查询 | XLEN XRANGE |
| 删除 | DEL         |

Stream支持按群组操作，Stream群组具备如下特性：
1.群组里可以多个成员，成员名称由消费方自定义，组内唯一即可。
2.同个群组共享消息，消息被其中一个消费者消费之后，其它消费者不会再重复消费。
3.不在群组内的客户端，也可以通过XREAD命令来和群组一起消费，即群组和非群组可以混用，这个
时候其实把不在群组的客户端理解为一个单独的群组

## 运行原理

### 内存存储

redis的数据库结构如下
```C
// Redis 7.0.9
// redis/src/server.h

/* Redis database representation. There are multiple databases identified
 * by integers from 0 (the default database) up to the max configured
 * database. The database number is the 'id' field in the structure. */
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *blocking_keys_unblock_on_nokey;   /* Keys with clients waiting for
                                             * data, and should be unblocked if key is deleted (XREADEDGROUP).
                                             * This is a subset of blocking_keys*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
    clusterSlotToKeyMapping *slots_to_keys; /* Array of slots to keys. Only used in cluster mode (db 0). */
} redisDb;
```

其中dict即为[[Code/7.Database/Database.0.Redis学习#HASHTABLE\|HASHTABLE]]中的hash table 结构，可见redis数据库中各个元素也以hash表的形式存储

### 单线程 | 多线程

> [Redis与Reactor模式 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1420724)
> [Redis 多线程网络模型全面揭秘 - 编程札记 - SegmentFault 思否](https://segmentfault.com/a/1190000039223696)

**Redis核心操作为单线程操作**，在辅助模块例如复制，网络I/O解包等使用多线程
**Redis 程序并不是单线程的**，Redis 在启动的时候会**启动后台线程（BIO）**:
- V2.6会启动 2 个后台线程，分别处理关闭文件、AOF 刷盘这两个任务
- V4.0之后新增了一个新的后台线程，用来异步释放 Redis 内存，也就是 lazyfree 线程

高速:
- Redis的大部分操作在内存上完成
- Redis的数据结构高效
- Redis是单线程的,省去了很多线程上下文切换的时间和消耗
- Redis 采用了多路复用机制，使其在网络IO操作中能并N发处理大量的客户端请求，实现高吞吐量

>多路指的是多个socket连接，复用指的是复用一个线程

原因:
redis的性能瓶颈主要在于内存大小和网络通信能力

优点:
- 没有并发带来的上下文切换成本和加锁成本

Redis服务器中有两类事件，文件事件和时间事件。
- 文件事件（file event）：Redis客户端通过socket与Redis服务器连接，而文件事件就是服务器对套接字操作的抽象。例如，客户端发了一个GET命令请求，对于Redis服务器来说就是一个文件事件。
- 时间事件（time event）：服务器定时或周期性执行的事件。例如，定期执行RDB持久化。

#### Unix系统调用

当如下任一情况发生时，会产生socket的**可读**事件：

- 该socket的接收缓冲区中的数据字节数大于等于socket接收缓冲区低水位标记的大小；
- 该socket的读半部关闭（也就是收到了FIN），对这样的socket的读操作将返回0（也就是返回EOF）；
- 该socket是一个监听socket且已完成的连接数不为0；
- 该socket有错误待处理，对这样的socket的读操作将返回-1。

当如下任一情况发生时，会产生socket的**可写**事件：

- 该socket的发送缓冲区中的可用空间字节数大于等于socket发送缓冲区低水位标记的大小；
- 该socket的写半部关闭，继续写会产生SIGPIPE信号；
- 非阻塞模式下，connect返回之后，该socket连接成功或失败；
- 该socket有错误待处理，对这样的socket的写操作将返回-1。

Unix提供`select`，`poll`，`kqueue`，`epoll`这样的系统调用，这些系统调用的功能是：你告知我一批socket，当这些socket的可读或可写事件发生时，我通知你这些事件信息

每一个socket都有对应的fd（即文件描述符)
`select`通过bitmap接收程序指定的fd集合，同样通过bitmap返回实际发生事件的fd集合，程序通过扫描返回的fd集合确认实际发生事件的fd
`poll`接收包含fd，关注事件和实际事件的`pollfd`结构的数组，系统通过`epoll`记录数组，当其中fd的事件发生时返回并修改相应`pollfd`中的实际事件

>![huasw2yqmn.jpeg (642×333) (qcloudimg.com)](https://ask.qcloudimg.com/http-save/yehe-1640143/huasw2yqmn.jpeg?imageView2/2/w/2560/h/7000)
**Handles** ：表示操作系统管理的资源，我们可以理解为fd。
**Synchronous Event Demultiplexer** ：同步事件分离器，阻塞等待Handles中的事件发生。
**Initiation Dispatcher** ：初始分派器，作用为添加Event handler（事件处理器）、删除Event handler以及分派事件给Event handler。也就是说，Synchronous Event Demultiplexer负责等待新事件发生，事件发生时通知Initiation Dispatcher，然后Initiation Dispatcher调用event handler处理事件。
**Event Handler** ：事件处理器的接口
**Concrete Event Handler** ：事件处理器的实际实现，而且绑定了一个Handle。因为在实际情况中，我们往往不止一种事件处理器，因此这里将事件处理器接口和实现分开，与C++、Java这些高级语言中的多态类似。
以上各子模块间协作的步骤描述如下：
>1.  我们注册Concrete Event Handler到Initiation Dispatcher中。
>2.  Initiation Dispatcher调用每个Event Handler的get_handle接口获取其绑定的Handle。
>3.  Initiation Dispatcher调用handle_events开始事件处理循环。在这里，Initiation Dispatcher会将步骤2获取的所有Handle都收集起来，使用Synchronous Event Demultiplexer来等待这些Handle的事件发生。
>4.  当某个（或某几个）Handle的事件发生时，Synchronous Event Demultiplexer通知Initiation Dispatcher。
>5.  Initiation Dispatcher根据发生事件的Handle找出所对应的Handler。
>6.  Initiation Dispatcher调用Handler的handle_event方法处理事件。

#### 单线程模型

redis中将socket设置为非阻塞式，使用I/O多路复用来处理socket连接

```C
int main(int argc, char **argv) {
	...
	initServer();
	...
	aeMain();
	...
	aeDeleteEventLoop(server.el);
	return 0;
}
```

>![](https://segmentfault.com/img/remote/1460000039223698)

- `initServer()`
  1. 使用`listenToPort`监听端口，创建监听socket
  2. . 调用`aeCreateFileEvent`绑定socket以及对应的接收处理函数`acceptTcpHandler`
- `aeMain()`- 执行事件处理循环
  1. 通过`epoll_wait`等待连接
  2. 连接到达时主线程使用 AE 的 API 将 `readQueryFromClient` 命令读取处理器绑定到新连接对应的fd上，并初始化一个 `client` 绑定这个客户端连接
  3. 客户端发送请求命令，触发读就绪事件，主线程调用 `readQueryFromClient` 通过 socket 读取客户端发送过来的命令存入 `client->querybuf` 读入缓冲区；
  4. 解析命令，检查命令可行性，执行命令
  5. 执行完成后`getGenericCommand`里使用`addReply`将结果放入client的输出缓冲区:`client->buf` 或者 `client->reply`，最后把 `client` 添加进一个 LIFO 队列 `clients_pending_write`
  6. 继续循环，在等待事件到来前的`beforeSleep`向客户端发送回包
- `aeDeleteEventLoop(server.el)`中关闭停止事件处理循环
- 退出

#### Redis多线程模型

redis在V6.0以后引入多线程以提升网络I/O的性能，默认关闭多线程模式，用户可在`redis.conf`中开启
```bash
io-threads 4 # 最大为128
io-threads-do-reads yes
```

> ![2016012682-44f6eb72869ece9c_fix732 (732×473) (segmentfault.com)](https://segmentfault.com/img/remote/1460000039223706)

与单线程模型的差异:
- `initServer()`中调用`initThreadedIO`来初始化多线程
- `readQueryFromClient`不读取客户端请求命令，仅调用`postponeClientRead`将client加入到`clients_pending_read`任务队列中去，主线程之后再分配 I/O 线程去读取客户端请求命令
- 在`beforeSleep`中执行`handleClientsWithPendingWritesUsingThreads`利用 Round-Robin 轮询负载均衡策略，把`clients_pending_read`队列中的连接均匀地分配给 I/O 线程各自的本地 FIFO 任务队列和主线程自己，I/O 线程通过 socket 读取客户端的请求命令，存入 `client->querybuf` 并解析第一个命令，**但不执行命令**，主线程忙轮询，等待所有 I/O 线程完成读取任务
- 主线程和所有 I/O 线程都完成了读取任务，主线程结束忙轮询，遍历 `clients_pending_read` 队列，**执行所有客户端连接的请求命令**，将响应数据写入到对应 `client` 的写出缓冲区，最后把 `client` 添加进一个 LIFO 队列 `clients_pending_write`
- 在`beforeSleep`中执行`handleClientsWithPendingWritesUsingThreads`分配`clients_pending_write` 队列，多线程向客户端发送回包

### 内存淘汰

#### 过期策略

> [【吊打面试】Redis的过期策略和内存淘汰策略不要搞混淆 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1643921)

过期策略通常有以下三种：
- **定时过期**: 每个设置过期时间的key都需要创建一个定时器，到过期时间就会立即清除
  - 可以立即清除过期的数据，对内存很友好
  - 但是会占用大量的CPU资源去处理过期的数据，从而影响缓存的响应时间和吞吐量。
- **惰性过期**: 只有当访问一个key时，才会判断该key是否已过期，过期则清除
  - 最大化地节省CPU资源，
  - 对内存非常不友好，极端情况可能出现大量的过期key没有再次被访问，从而不会被清除，占用大量内存
- **定期过期**: 每隔一定的时间，会扫描一定数量的数据库的expires字典中一定数量的key，并清除其中已过期的key。

Redis同时使用**惰性过期和定期过期**两种过期策略。但是Redis定期删除是随机抽取机制，不可能扫描删除掉所有的过期Key。因此需要内存淘汰机制

String对象可通过SET方式设置过期时间
- SET key value EX seconds
- SET key value PX milliseconds
- TTL key - 查看剩余时间

通用方法：
- EXPIRE key seconds
- PEXPIRE key milliseconds

#### 内存淘汰策略

Redis可通过`redis.conf`中的`maxmemory`设置内存最大占用，超过时触发内存数据淘汰

淘汰策略共八种
- `noeviction`(default): 不淘汰，内存超过限制之后所有的写入操作都失败并返回错误，但读操作正常进行
- `volatile-lru`: 最近最少使用算法，从设置了过期时间的键中选择最久未使用的键值对清除掉
- `volatile-lfu`: 最近最不经常使用算法，从设置了过期时间的键中选择某段时间之内使用频次最小的键值对清除掉
- `volatile-ttl`: 从设置了过期时间的键中选择存活时间最短的键值对清除
- `volatile-random`: 从设置了过期时间的键中随机选择键进行清除
- `allkeys-lru`: 从所有的键中选择最久未使用的键值对清除
- `allkeys-lfu`: 从所有的键中选择某段时间之内使用频次最少的键值对清除
- `allkeys-random`: 从所有的键中随机选择键进行删除

淘汰触发: 每次运行读写命令的时候

##### Redis-LRU

LRU为淘汰最久未使用的key，做法是为所有数据维护一个顺序链表，但数据多时成本巨大，所以Redis选择采样的方式来做，也就是近似LRU算法

在LRU模式，redisObject对象中lru字段存储的是key被访问时Redis的时钟server.lrulock，当key被访问的时候，Redis会更新这个key的redisObject的lru字段

>注意，Redis为了保证核心单线程服务性能，缓存了Unix操作系统时钟，默认每毫秒更新一次，缓存的值是Unix时间戳取模2^24

近似 LRU 算法在现有数据结构的基础上采用随机采样的方式来淘汰元素，当内存不足时，就执行一次近似 LRU 算法。具体步骤是随机采样 n 个 key(默认为 5)然后根据时间戳淘汰掉最旧的那个 key，如果淘汰后内存还是不足，就继续随机采样来淘汰

V3.0后redis维护一个大小为16的候选池，池中的数据根据访问时间进行排序，首次随机选取的key都放入池中，之后只将选取key中活性低于池中活性最小值的key放入池中，每次放入后都删除池中活性最小的key

##### Redis-LFU

LFU优先淘汰使用频率最低的key

Redis在LFU策略下复用`redisObject.lru`字段，高16bit存储ldt(上次访问时间戳)，低8bit存储logc(访问次数)，默认情况下新key的访问次数为5，防止其因为访问频率过低而直接被删除

```C
/* Update LFU when an object is accessed.
 * Firstly, decrement the counter if the decrement time is reached.
 * Then logarithmically increment the counter, and update the access time. */
void updateLFU(robj *val) {
    unsigned long counter = LFUDecrAndReturn(val);
    counter = LFULogIncr(counter);
    val->lru = (LFUGetTimeInMinutes()<<8) | counter;
}
```

1. 计算次数衰减
2. 一定概率增加访问次数 - 次数不足5次则一定增加，如果大于5次，小于255次，会一定概率加1，原来的次数越大，越困难，最大为255次
3. 更新时间戳

## 持久化

Redis在内存中运行，通过将数据保存到存储设备中实现持久化

Redis提供两种持久化方法:
- RDB(Redis Database)，记录Redis某个时刻的全部数据，本质是数据快照，直接保存二进制数据到磁盘，后续通过加载RDB文件恢复数据
- AOF(Append Only File)，记录执行的每条命令，重启之后通过重放命令来恢复数据，本质是记录操作日志，后续通过日志重放恢复数据

体积方面：相同数据量下，RDB体积更小，因为RDB记录的是二进制紧凑型数据
恢复速度：RDB是数据快照，可以直接加载，而AOF文件恢复，相当于重放情况，RDB显然会更快
数据完整性：AOF记录了每条日志，RDB是间隔一段时间记录一次，用AOF恢复数据通常会更为完整

同时开启RDB和AOF时优先使用AOF恢复数据，但Redis官方不建议单独开AOF

### RDB

```shell
# /etc/redis/redis.conf
save 3600 1 300 100 60 10000

dbfilename dump.rdb # 文件名

dir ./ # 位置
```
每间隔3600s，发生至少1次写数据操作，就执行RDB，以此类推

RDB本质是Redis的数据快照，这种方式是最常见、最稳定的数据持久化手段
Redis中RDB的触发方式有四种
- 达到阈值周期函数触发
- 正常关闭Redis服务触发
- 主动执行BGSAVE命令触发
- 主动执行SAVE命令触发 - 不fork子进程而由主进程进行RDB，会阻塞主进程

Redis通常是通过fork一个子进程的方式来进行RDB，配合写时复制技术，相当于异步执行，和主进程互不干扰，将对执行流程的影响降到最低

### AOF

```shell
# /etc/redis/redis.conf
appdendonly no # 是否开启AOF

appendenfilename "appendonly.aof"

# appendfsync always # 每次请求都刷入AOF
appendfsync everysec # 每秒刷入一次
# appendfsync no     # 不主动刷入，一般Linux系统每30秒刷入一次
```

写入步骤:
1. 将数据写入`sds`结构的`aof_buf`中
2. `aof_buf`对应数据写入磁盘缓冲区
3. 写入磁盘

Redis可以在AOF文件体积变得过大时，自动地在后台fork一个子进程对AOF进行重写，合并针对相同Key的操作
在重写过程中发生的新的操作记录同时记录在原有的AOF缓冲区和AOF重写缓冲区，新AOF文件创建完毕后Redis将重写缓冲区内容追加到新的AOF文件，再用新AOF文件替换原来的AOF文件

![image.png](https://image.jiang849725768.asia/2023/202303131715428.png)

重写触发条件配置:
```shell
# /etc/redis/redis.conf
# 相比上次重写时候数据增长100%
auto-aof-rewrite-percentage 100
# 文件大小超过指定值
auto-aof-rewrite-min-size 64mb
```

#### MP-AOF

RedisV7.0采用了新的AOF重写方案MP-AOF(多部件AOF)
MP-AOF将AOF文件分为 BASE AOF和INCR AOF两个，发生重写时redis新建一个INCR AOF文件以供aof_buf书写，而将旧BASE AOF以及INCR AOF合并重写为新的BASE AOF，此外当前有效的BASE AOF以及INCR AOF通过manifest文件记录，重写完成后更新该文件

### 混合持久化

混合持久化在AOF重写阶段中将当前状态保存为RDB二进制文件写入AOF文件，再将重写缓冲区内容追加至AOF文件
引入了混合持久化之后，使用 AOF 重建数据集时，会通过文件开头是否为“REDIS”来判断是否为混合持久化

![c1618be1f307ae6abe68549b5831ee7f.png (431×761) (csdnimg.cn)](https://img-blog.csdnimg.cn/img_convert/c1618be1f307ae6abe68549b5831ee7f.png)
