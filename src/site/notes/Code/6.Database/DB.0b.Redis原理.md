---
{"dg-publish":true,"permalink":"/Code/6.Database/DB.0b.Redis原理/","title":"Redis 原理","noteIcon":""}
---


# Redis 原理

## 单线程 | 多线程

> [Redis与Reactor模式 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1420724)
> [Redis 多线程网络模型全面揭秘 - 编程札记 - SegmentFault 思否](https://segmentfault.com/a/1190000039223696)

**Redis 核心操作为单线程操作**，在辅助模块例如复制，网络 I/O 解包等使用多线程
**Redis 程序并不完全是单线程**，Redis 启动时会**启动后台线程(BIO)**:
- V2.6 会启动 2 个后台线程，分别处理关闭文件、AOF 刷盘这两个任务
- V4.0 之后新增了一个新的后台线程，用来异步释放 Redis 内存，也就是 lazyfree 线程

单线程优点：没有并发带来的上下文切换成本和加锁成本
多线程原因：redis 的性能瓶颈主要在于内存大小和网络通信能力

Redis 服务器中有两类事件，文件事件和时间事件。
- 文件事件(file event)：Redis 客户端通过 socket 与 Redis 服务器连接，而文件事件就是服务器对套接字操作的抽象。例如，客户端发了一个 GET 命令请求，对于 Redis 服务器来说就是一个文件事件
- 时间事件(time event)：服务器定时或周期性执行的事件。例如，定期执行 RDB 持久化

### 单线程模型

redis 中将 socket 设置为非阻塞式，使用 I/O 多路复用来处理 socket 连接

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
  1. 使用 `listenToPort` 监听端口，创建监听 socket
  2. . 调用 `aeCreateFileEvent` 绑定 socket 以及对应的接收处理函数 `acceptTcpHandler`
- `aeMain()` - 执行事件处理循环
  1. 通过 `epoll_wait` 等待连接
  2. 连接到达时主线程使用 AE 的 API 将 `readQueryFromClient` 命令读取处理器绑定到新连接对应的 fd 上，并初始化一个 `client` 绑定这个客户端连接
  3. 客户端发送请求命令，触发读就绪事件，主线程调用 `readQueryFromClient` 通过 socket 读取客户端发送过来的命令存入 `client->querybuf` 读入缓冲区；
  4. 解析命令，检查命令可行性，执行命令
  5. 执行完成后 `getGenericCommand` 里使用 `addReply` 将结果放入 client 的输出缓冲区: `client->buf` 或者 `client->reply`，最后把 `client` 添加进一个 LIFO 队列 `clients_pending_write`
  6. 继续循环，在等待事件到来前的 `beforeSleep` 向客户端发送回包
- `aeDeleteEventLoop(server.el)` 中关闭停止事件处理循环
- 退出

### Redis 多线程

redis 在 V6.0以后引入多线程以提升网络 I/O 的性能，默认关闭多线程模式，用户可在 `redis.conf` 中开启
```bash
io-threads 1 # 启用线程数，最大为128，改动后仅启用写多线程 
io-threads-do-reads yes # 允许读多线程
```

Redis 6.0 引入了多线程 I/O，它的总体设计思路是：
- 主线程接受客户端并创建连接，注册读事件，读事件就绪时，主线程将读事件放到一个队列中，这个队列的顺序代表收到客户端请求的顺序，所以意味着 redis 也要按这个队列顺序执行命令
- 主线程利用 RR 策略，将读事件分配给多个 I/O 线程，然后主线程开始忙等待(等待包括主线程在内的 I/O 多线程读取完成)
  - I/O 线程读取数据到输入缓冲区，并解析命令（不执行)
- 主线程忙等待结束，单线程执行解析后的命令，将响应写入输出缓冲区
- 返回响应时主线程同样利用 RR 策略，将写事件分配给多个 I/O 线程，然后主线程开始忙等待(等待包括主线程在内的 I/O 多线程写出完成)
总结一句话就是：**多线程读取/写入，单线程执行命令**

> ![2016012682-44f6eb72869ece9c_fix732 (732×473) (segmentfault.com)|600](https://segmentfault.com/img/remote/1460000039223706)

与单线程模型的差异:
- `initServer()` 中调用 `initThreadedIO` 来初始化多线程
- `readQueryFromClient` 不读取客户端请求命令，仅调用 `postponeClientRead` 将 client 加入到 `clients_pending_read` 任务队列中去，主线程之后再分配 I/O 线程去读取客户端请求命令
- 在 `beforeSleep` 中执行 `handleClientsWithPendingWritesUsingThreads`，利用 Round-Robin 轮询负载均衡策略，把 `clients_pending_read` 队列中的连接均匀地分配给 I/O 线程各自的本地 FIFO 任务队列和主线程自己，I/O 线程通过 socket 读取客户端的请求命令，存入 `client->querybuf` 并解析第一个命令，**但不执行命令**，主线程忙轮询，等待所有 I/O 线程完成读取任务
- 主线程和所有 I/O 线程都完成了读取任务，主线程结束忙轮询，遍历 `clients_pending_read` 队列，**执行所有客户端连接的请求命令**，将响应数据写入到对应 `client` 的写出缓冲区，最后把 `client` 添加进一个 LIFO 队列 `clients_pending_write`
- 在 `beforeSleep` 中执行 `handleClientsWithPendingWritesUsingThreads` 分配 `clients_pending_write` 队列，多线程向客户端发送回包

## 过期删除&内存淘汰

### 过期删除策略

> [【吊打面试】Redis的过期策略和内存淘汰策略不要搞混淆 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1643921)

#### 过期时间设置

在创建时设置过期时间:
- `set {key} {value} ex {seconds}`
- `set {key} {value} px {milliseconds}`
- `setex {key} {seconds} {value}`

通用方法：
- `EXPIRE {key} seconds`
- `PEXPIRE {key} milliseconds`
- `TTL {key}` - 查看剩余时间
- `PERSIST {key}` - 取消过期时间

#### 过期判定

Redis 数据库结构中存在一个 hash table 形式的过期字典，当一个 key 设置了过期时间时，在过期字典内存储 `{key}-{expire time}`  键值对

> ![过期字典数据结构.png (1080×793) (xiaolincoding.com)|600](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E8%BF%87%E6%9C%9F%E7%AD%96%E7%95%A5/%E8%BF%87%E6%9C%9F%E5%AD%97%E5%85%B8%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png)

当查询一个 key 时，Redis 首先查询过期字典判定该 key 是否过期

#### 过期删除

常见的过期删除策略有以下三种：
- **定时删除**: 每个设置过期时间的 key 都需要创建一个定时器，到过期时间就会立即清除
  - 可以立即清除过期的数据，对内存友好
  - 会占用 CPU 资源去处理过期的数据，从而影响缓存的响应时间和吞吐量
- **惰性删除**: 只有当访问一个 key 时，才会判断该 key 是否已过期，过期则清除
  - 最大化地节省 CPU 资源
  - 对内存不友好，极端情况可能出现大量的过期 key 没有再次被访问，从而不会被清除，占用大量内存
- **定期删除**: 每隔一定的时间，随机扫描一定数量的 key，清除其中已过期的 key
  - 减少了删除操作对 CPU 的影响，同时一定程度上避免了过期键对空间的无效占用
  - 随机扫描的数量难以确定在一个平衡点

**Redis 同时使用惰性删除和定期删除两种过期策略**
- Redis 在访问或者修改 key 之前，都会调用 `expireIfNeeded` 函数对其进行检查，检查 key 是否过期
- Redis 每隔一段时间(default: 0.1s)随机**从过期字典中**取出一定数量(20)的 key 进行检查，并删除其中的过期 key

定期删除会阻塞命令执行，因此会设置删除执行时间上限，保证不会造成过长时间停顿

由于 Redis 定期删除是随机抽取机制，不可能扫描删除掉所有的过期 Key，因此需要*内存淘汰*机制

### 内存淘汰策略

Redis 可通过 `redis.conf` 中的 `maxmemory` 参数设置内存最大占用，每次进行读写的时候检查是否触发*内存数据淘汰*
`maxmemory` 在 64 位系统中默认为 0，不限制最大内存占用，但当 Redis 内存超出物理内存的限制时，内存的数据会开始和磁盘产生频繁的 swap，让 Redis 的性能急剧下降

Redis 的内存淘汰策略共八种

不淘汰：
- `noeviction` (default): 不淘汰，内存超过限制之后写入操作会触发 OOM(Out Of Memory)，但没有新写入时读操作和删除操作正常进行

仅从过期字典中淘汰：
- `volatile-lru`: 最近最少使用算法，从设置了过期时间的键中选择**最久未使用**的键值对清除
- `volatile-lfu`: 最近最不经常使用算法，从设置了过期时间的键中选择**某段时间之内使用频次最小**的键值对清除
- `volatile-ttl`: 从设置了过期时间的键中选择**存活时间最短**的键值对清除
- `volatile-random`: 从设置了过期时间的键中**随机选择**键清除

全局淘汰：
- `allkeys-lru`: 从所有的键中选择**最久未使用**的键值对清除
- `allkeys-lfu`: 从所有的键中选择**某段时间之内使用频次最少**的键值对清除
- `allkeys-random`: 从所有的键中**随机选择**键进行删除

内存淘汰策略修改方式：
- 通过 `config set maxmemory-policy {policy}` 命令设置，设置之后立即生效，不需要重启 Redis 服务，**重启 Redis 后设置失效**
- 通过 `redis.conf` 中 `maxmemory-policy` 参数设置，**重启 Redis 服务后**设置开始永久生效

#### Redis-LRU

LRU 优先淘汰**最久未使用**的 key

常规 LRU 算法为所有数据维护一个顺序链表，带来额外的空间开销，同时大量数据被访问时链表移动操作过多，降低性能

Redis 采用近似 LRU 算法，在现有数据结构的基础上采用随机采样的方式来淘汰元素，当内存不足时，就执行一次近似 LRU 算法
- 随机采样 n 个 key(default: 5)
- 根据时间戳淘汰掉最旧的那个 key
- 如果淘汰后内存还是不足，就继续随机采样来淘汰

在 LRU 模式，`redisObject.lru` 字段(24 bytes)存储的是 key 被访问时 Redis 的时钟 `server.Lrulock`，当 key 被访问的时候，Redis 会更新对应 redisObject 的 lru 字段

>注意，Redis 为了保证核心单线程服务性能，缓存了 Unix 操作系统时钟，默认每毫秒更新一次，缓存的值是 Unix 时间戳取模2^24

V3.0 后 redis 维护一个大小默认为 16 的候选池，池中的数据根据访问时间进行排序，首次随机选取的 key 都放入池中，之后只将选取 key 中活性低于池中活性最小值的 key 放入池中，每次放入后删除池中活性最小的 key

**LRU 无法解决缓存污染问题**：应用一次读取了大量的数据，而这些数据只会被读取这一次，那么这些数据会留存在 Redis 缓存中很长一段时间，造成缓存污染

#### Redis-LFU

LFU 优先淘汰**最近最不常用**的 key，V4.0 引入

Redis 在 LFU 策略下复用 `redisObject.lru` 字段，高 16bit 存储 *ldt*(上次访问时间戳)，低 8bit 存储 *logc*(访问次数)，默认情况下新添加 key 的访问频次为 5，防止 key 过早被删除

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

1. 按照上次访问距离当前的时长对 logc 进行衰减
2. 将 logc 按一定概率增加
    - 次数不足 5 次则一定增加
    - 大于 5 次，小于 255 次，会一定概率加 1，原来的次数越大，越困难
    - 最大为 255 次
3. 更新时间戳

## 持久化

Redis 在内存中运行，通过将数据保存到存储设备中实现持久化

Redis 提供两种持久化方法:
- *RDB*(Redis Database)，记录 Redis 某个时刻的全部数据，本质是数据快照，直接保存二进制数据到磁盘，后续通过加载 RDB 文件恢复数据
- *AOF*(Append Only File)，记录**执行的每条写操作命令**，重启之后通过重放命令来恢复数据，本质是记录操作日志，后续通过日志重放恢复数据

### RDB VS AOF

*体积方面*：相同数据量下，RDB 体积更小，因为 RDB 记录的是二进制紧凑型数据
*恢复速度*：RDB 是数据快照，可以直接加载，而 AOF 文件恢复，相当于重放情况，RDB 更快
*数据完整性*：AOF 记录了每条日志，RDB 间隔一段时间记录一次，用 AOF 恢复数据通常会更为完整

同时开启 RDB 和 AOF 时优先使用 AOF 恢复数据，但 Redis 官方不建议单独开 AOF

### RDB

```shell
# /etc/redis/redis.conf
save 3600 1 300 100 60 10000

dbfilename dump.rdb # 文件名

dir ./ # 位置
```

`save` 参数配置执行 BGSAVE 命令进行 RDB 的阈值
`save 3600 1` 表示每间隔 3600s，发生至少 1 次写数据操作，就执行 RDB，以此类推

RDB 本质是 Redis 的数据快照，这种方式是最常见、最稳定的数据持久化手段

Redis 中 RDB 的*触发方式*有四种：
- 操作达到设置的阈值时周期函数触发
- 正常关闭 Redis 服务触发
- 主动执行 BGSAVE 命令触发
- 主动执行 SAVE 命令触发 - 不 fork 子进程而由主进程进行 RDB，会阻塞主进程

Redis 通过 `fork()` 创建一个子进程来执行 BGSAVE，配合[[Code/4.OperatingSystem/OS.0b.内存管理#写时复制\|写时复制技术]]，相当于异步执行，和主进程互不干扰，将对执行流程的影响降到最低

根据写时复制的性质，Redis 主进程在 RDB 期间进行的新的写操作**不被记录在快照中**，且会**导致被修改数据的物理内存的复制**
因此主进程在 RDB 期间对大 key 的写操作会导致内核花费较长时间复制物理内存，阻塞主进程

### AOF

```shell
# /etc/redis/redis.conf
appdendonly no # 是否开启AOF
appendenfilename "appendonly.aof"
```

Redis 每次执行写操作命令后将该命令记录至 AOF 日志

写入步骤：
1. 将数据写入 `sds` 结构的 `server.aof_buf` 中
2. 通过 `write()` 系统调用，将 `aof_buf` 对应数据写入内核缓冲区 PageCache
3. 内核将数据写入磁盘

AOF 提供了*always*、*everysec*、*no*三种写入磁盘的策略：
```shell
# appendfsync always # 每次记录命令后立刻写回磁盘
appendfsync everysec # 每秒将缓冲区刷入磁盘
# appendfsync no     # 不主动刷入，一般Linux系统每30秒刷入一次
```

- Always 策略为每次写入 AOF 文件数据后，主线程立刻执行 fsync() 函数
- (default)Everysec 策略为每秒**创建一个子线程异步执行** fsync() 函数
- No 策略就是**程序本身**永不执行 fsync() 函数(一般 Linux 系统每 30 秒刷入一次)

3 种写回策略都**无法完美解决*主进程阻塞*和*减少数据丢失*的平衡问题**
- 由于写操作命令执行成功后才记录到 AOF 日志，所以不会阻塞当前写操作命令的执行，但是执行 fsync() 函数**可能会给下一个命令带来阻塞风险**
- 当 Redis 在还没来得及将命令写入到硬盘时，服务器发生宕机了，**数据就会有丢失的风险**

#### AOF 重写

AOF 日志文件大小随写操作命令增加而增大，Redis 可以在 AOF 日志体积变得过大时，自动地在后台创建一个**子进程** *bgrewriteaof* 重写 AOF
重写机制合并针对相同 Key 的操作，根据键值对当前的最新状态用一条命令记录键值对，**生成一个压缩后的 AOF 日志文件替代原文件**

>子进程进行 AOF 重写期间，主进程可以继续处理命令请求，从而避免阻塞主进程
>子进程带有主进程的数据副本，且不像线程需要通过加锁来保证共享数据的安全，导致降低性能
>采用进程的方式的隔离性更好，如果线程崩溃会带来整个进程的崩溃，而子进程的崩溃对父进程不造成影响，提高在 AOF 日志重写和 RDB 快照生成时的安全性

子进程通过[[Code/4.OperatingSystem/OS.0b.内存管理#写时复制\|写时复制技术]]和父进程共享物理内存数据

重写过程中发生的新的写操作记录同时记录在 *AOF 缓冲区*和 *AOF 重写缓冲区*
> [!TODO] **TODO** 新 AOF 文件创建完毕后
> - 以前的版本是主进程直接将 aof 重写缓冲区的数据写到新的 aof 文件
> - 后面用管道优化为主进程将数据通过管道传递给子进程，子进程将其写入磁盘，要是子进程结束后 aof 重写 buf 中还有剩余数据，那就是主进程收尾将其写入新的 aof 文件中

重写缓冲区避免了父子进程写入同一文件而竞争文件系统锁进而对 Redis 主线程的性能造成影响

![image.png|600](https://image.jiang849725768.asia/2023/202303131715428.png)

重写触发条件:
```shell
# /etc/redis/redis.conf
# 相比上次重写时候数据增长100%
auto-aof-rewrite-percentage 100
# 文件大小超过指定值
auto-aof-rewrite-min-size 64mb
```

##### 阻塞

- 创建子线程过程中调用 fork 函数做内存拷贝的时候如果需要拷贝的数据量很大有可能会发生阻塞
- 父进程修改了共享数据，就会发生写时复制，对 bigkey 的拷贝也会发生阻塞
- 主进程调用信号处理函数，将 aof 重写缓冲区内的命令写入到新 aof 文件中

#### MP-AOF

RedisV7.0 采用了新的 AOF 重写方案 MP-AOF(多部件 AOF)
MP-AOF 将 AOF 文件分为 *BASE AOF* 和 *INCR AOF* 两个文件，发生重写时 redis **将旧 BASE AOF 以及 INCR AOF 合并**重写为新的 BASE AOF，同时新建一个 INCR AOF 文件以供 aof_buf 书写
Redis 通过 *manifest* 文件记录当前有效的 BASE AOF 以及 INCR AOF ，重写完成后通过更新该文件记录更新 AOF

MP-AOF 主要的意义在于减少了 aof rewrite buffer 的内存开销，以及发送和写入 aof rewrite buffer 时的 CPU 与 IO 开销

#### 宕机

数据丢失：
Always 会丢失一个时间循环(epoll wait)里面的所有命令
Everysec 策略下可能会丢失 2s 的数据，因为如果当主线程尝试进行 `write()` 系统调用时发现 2s 内的上一次子线程的刷盘还未结束，主线程会跳过该次写入以避免阻塞，但如果超过 2s 则主线程会阻塞等待上一子线程刷盘结束

重写宕机：
V7.0 以前宕机恢复后可以使用旧 AOF 文件恢复数据
V7.0 以后使用旧 base aof + 旧 incr aof + 新 incr aof 文件恢复数据

### 混合持久化

V4.0 提出，开启方式：
```shell
aof-use-rdb-preamble yes
```

混合持久化**作用于 AOF 重写阶段**，AOF 重写日志时，重写子进程将**与父进程共享的内存数据**以 *RDB 二进制形式*写入 AOF 文件，再将重写缓冲区内容(重写过程中的新写操作命令)追加至 AOF 文件

> ![c1618be1f307ae6abe68549b5831ee7f.png (431×761) (csdnimg.cn)](https://img-blog.csdnimg.cn/img_convert/c1618be1f307ae6abe68549b5831ee7f.png)

引入了混合持久化之后，使用 AOF 重建数据集时，会通过文件开头是否为“REDIS”来判断是否为混合持久化

>[!Tip] 内存大页影响 redis 性能
>Linux 内核从 2.6.38 开始支持内存大页机制，该机制支持 2MB 大小的内存页分配，而常规的内存页分配是按 4KB 的粒度来执行的
>采用了内存大页，那么即使客户端请求只修改 100B 的数据，在发生写时复制后，Redis 也需要拷贝 2MB 的大页

## 事务

Redis 原生事务包含 *MULTI*，*EXEC*，*DISCARD* 和 *WATCH* 四个命令
MULTI 开启事务，开始输入事务指令
EXEC 执行事务
DISCARD 在执行前取消事务
WATCH 可以监控一个或多个键，一旦其中有一个键被修改(或删除)，阻止之后的事务执行
e.g.
```shell
WATCH mykey
val = GET mykey
val = val + 1
MULTI
SET mykey $val
EXEC
```

原理：redis 存储了一个包含命令队列的结构体，顺序执行任务

**Redis 事务不具备原子性**，仅能通过单线程的特性使得事务执行过程中其他操作无法执行

在 EXEC 命令前的*语法错误*命令会导致事务不执行直接返回错误，在事务执行时*运行错误*的命令不影响其他命令的执行

问题：
- Watch 难用
- 事务中间失败时不停止

### Lua 事务

V2.6 后 Redis 通过内嵌支持 Lua 环境，通过*EVAL*命令原子性执行脚本

优点：
- 可以编写 `if else` 逻辑
- 事务中间失败中断后续执行
- 不需要 Watch 保证执行状态未改变

**原子性保证**：
Redis 使用相同的 Lua 解释器来运行所有的命令
Redis 保证脚本以原子方式执行：在执行脚本时，不会执行其他脚本或 Redis 命令，从所有其他客户端的角度来看，脚本的效果要么仍然不可见，要么已经完成
>[!attention]
>如果 Lua 执行出错，可能就会出现一部分命令执行，一部分没有执行
