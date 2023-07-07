---
{"dg-publish":true,"permalink":"/Code/4.OS/OS.2.Linux网络系统/","title":"Linux 网络系统","noteIcon":""}
---


# Linux 网络系统

## 网络包接收发送

> Linux 网络协议栈
> ![|400](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E6%B5%AE%E7%82%B9/%E5%8D%8F%E8%AE%AE%E6%A0%88.png)

> ![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E6%B5%AE%E7%82%B9/%E6%94%B6%E5%8F%91%E6%B5%81%E7%A8%8B.png)

### 接收流程

1. 网卡接收到一个网络包后，通过 DMA 技术将网络包写入到内存中指定的 Ring Buffer 环形缓冲区，然后告诉操作系统这个网络包已经到达
2. 网络包进入到网络接口层，检查报文的合法性，不合法则丢弃，合法则找出该网络包的上层协议类型(IPv4/IPv6)，去掉帧头和帧尾，然后交给网络层
3. 网络层取出 IP 包判断下一步的走向，当确认这个网络包发送给本机后，找出上一层协议的类型(TCP/UDP)，接着去掉 IP 头交给传输层
4. 传输层取出 TCP 头或 UDP 头，根据四元组(源 IP、源端口、目的 IP、目的端口) 作为标识，找出对应的 Socket，并把数据放到 Socket 的接收缓冲区
5. 应用层程序调用 Socket 接口，将内核的 Socket 接收缓冲区的数据**拷贝**到应用层的缓冲区，然后唤醒用户进程

#### NAPI 机制

Linux 内核在 2.6 版本中引入了 *NAPI 机制*，混合「中断和轮询」的方式来接收网络包
NAPI 机制核心概念是**不采用中断的方式读取数据**，而是首先采用中断唤醒数据接收的服务程序，然后使用 `poll` 的方法来轮询数据
1. 网卡向 CPU 发起*硬件中断*，当 CPU 收到硬件中断请求后，根据中断表，调用已经注册的中断处理函数
2. 中断处理函数先暂时屏蔽中断，表示已经知道内存中有数据了，告诉网卡下次再收到数据包直接写内存就可以了，不要再通知 CPU 了，这样可以提高效率，避免 CPU 不停的被中断
3. 中断处理函数发起*软中断*，然后恢复刚才屏蔽的中断
4. 内核中的 ksoftirqd 线程专门负责软中断的处理，当 ksoftirqd 内核线程收到软中断后，就会来轮询处理数据，从 Ring Buffer 中获取一个数据帧，用 sk_buff 表示，从而可以作为一个网络包交给网络协议栈进行逐层处理

### 发送流程

1. 应用程序进行系统调用发送数据包的 Socket 接口，从用户态陷入到内核态中的 Socket 层，内核会申请一个内核态的 sk_buff 内存，将用户待发送的数据拷贝到 sk_buff 内存，并将其加入到发送缓冲区
2. 网络协议栈从 Socket 发送缓冲区中取出 sk_buff，并按照 TCP/IP 协议栈从上到下逐层处理
3. 如果使用 TCP 协议，**先拷贝一个新的 sk_buff 副本以备丢失重传**，然后对 sk_buff 填充 TCP 头
4. 网络层选取路由(确认下一跳的 IP)，填充 IP 头，netfilter 过滤，对超过 MTU 大小的数据包进行分片
5. 网络接口层通过 ARP 协议获得下一跳的 MAC 地址，然后对 sk_buff 填充帧头和帧尾，接着将 sk_buff 放到网卡的发送队列中
6. 触发*软中断*通知网卡驱动程序，驱动程序从发送队列中读取 sk_buff 并挂到 RingBuffer 中，接着将 sk_buff 数据映射到网卡可访问的内存 DMA 区域，最后触发真实的发送
7. 发送完成时网卡设备触发一个*硬中断*来释放内存，主要是释放 sk_buff 内存和清理 RingBuffer 内存
8. 收到这个 TCP 报文的 ACK 应答时，传输层释放原始的 sk_buff

**全部数据包只用一个结构体 sk_buff 表示**，通过调整 sk_buff 中 `data` 的指针填充或去掉首部
- 当接收报文时，从网卡驱动开始，通过协议栈层层往上传送数据报，通过增加 skb->data 的值，来逐步剥离协议首部
- 当要发送报文时，创建 sk_buff 结构体，数据缓存区的头部预留足够的空间，用来填充各层首部，在经过各下层协议时，通过减少 skb->data 的值来增加协议首部

**发送网络数据的时候涉及最多三次内存拷贝操作**
- 调用发送数据的系统调用的时候，内核会申请一个内核态的 sk_buff 内存，将用户待发送的数据拷贝到 sk_buff 内存，并将其加入到发送缓冲区
- 在使用 TCP 传输协议的情况下，从传输层进入网络层的时候，每一个 sk_buff 都会拷贝
- 当 IP 层发现 sk_buff 大于 MTU 时会再申请额外的 sk_buff，并将原来的 sk_buff 拷贝为多个小的 sk_buff

## 连接建立

在 TCP 三次握手过程中
1. 当服务器收到客户端的 SYN 包后，内核会把该连接存储到半连接队列，然后再向客户端发送 SYN+ACK 包
2. 接着客户端会返回 ACK，服务端收到第三次握手的 ACK 后，内核会把连接从半连接队列移除，然后创建新的完全的连接，并将其增加到全连接队列，等待进程调用 `accept()` 函数时取出连接

> ![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/TCP-%E5%8D%8A%E8%BF%9E%E6%8E%A5%E5%92%8C%E5%85%A8%E8%BF%9E%E6%8E%A5/3.jpg)



## IO 模型

常见的五种 I/O 模型如下：
1. 阻塞式 I/O 模型：在进行 I/O 操作时，应用程序会一直等待直到操作完成，期间无法进行其他操作。
2. 非阻塞式 I/O 模型：在进行 I/O 操作时，应用程序可以立即返回并继续执行其他操作，但需要不断轮询操作是否完成，这种方式也称为轮询(Polling)。
3. I/O 多路复用模型：通过 select、poll、epoll 等系统调用来同时监听多个 I/O 事件，当有事件发生时，通知应用程序进行处理。
4. 信号驱动式 I/O 模型：应用程序通过系统调用注册一个信号处理函数，当 I/O 操作完成时，操作系统会向应用程序发送一个信号，应用程序在信号处理函数中进行处理
5. 异步 I/O 模型：应用程序发起 I/O 操作后，可以立即返回并继续执行其他操作，当 I/O 操作完成后，操作系统会通知应用程序进行处理，这种方式也称为回调(Callback)

## IO 多路复用

> [9.2 I/O 多路复用：select/poll/epoll | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/os/8_network_system/selete_poll_epoll.html#select-poll)

### 基础Socket 模型

服务端
1. 调用 `socket()` 函数，创建网络协议为 IPv4，以及传输协议为 TCP 的监听 Socket ，接着调用 `bind()` 函数，给这个 Socket 绑定一个 **IP 地址和端口**
2. 调用 `listen()` 函数进行监听
3. 调用 `accept()` 函数从内核获取客户端的连接
    - 如果没有客户端连接，则会阻塞等待客户端连接的到来
    - 当 TCP 全连接队列不为空后从内核中的 TCP 全连接队列里拿出一个已经完成连接的 Socket 返回应用程序，后续数据传输都用这个连接 Socket

客户端
1. 创建 Socket
2. 调用 `connect()` 函数向指定 IP 地址和端口号的服务端发起连接

**监听的 Socket 和真正用来传数据的 Socket 是两个不同的 socket**

连接建立后，客户端和服务端开始相互传输数据，双方都可以通过 `read()` 和 `write()` 函数来读写数据

在内核中 Socket 也是**以文件的形式存在的**，也是有对应的文件描述符

基础 socket 模型基本只能一对一通信，因为使用的是同步阻塞的方式，当服务端在还没处理完一个客户端的网络 I/O 时，或者读写操作发生阻塞时，其他客户端是无法与服务端连接的

### 多进程/多线程模型

*多进程模型*为每个客户端分配一个进程来处理请求

服务器的主进程负责监听客户的连接，一旦与客户端连接完成，accept() 函数就会返回一个*已连接 Socket*，这时就通过 `fork()` 函数创建一个子进程
子进程把父进程所有相关的东西都**复制**一份，包括文件描述符、内存地址空间、程序计数器、执行代码等
子进程通过复制得到的文件描述符直接使用已连接 Socket 和客户端进行通信

子进程退出时内核里保留的该进程的信息占用内存，未回收就会变成*僵尸进程*，随着僵尸进程越多，会慢慢耗尽系统资源

进程的上下文切换不仅包含了虚拟内存、栈、全局变量等用户空间的资源，还包括了内核堆栈、寄存器等内核空间的资源，切换开销很大

*多线程模型*为每个客户端分配一个线程来处理请求

当服务器与客户端 TCP 完成连接后，通过 `pthread_create()` 函数创建线程，然后将*已连接 Socket*的文件描述符传递给线程函数，接着在线程里和客户端进行通信，从而达到并发处理的目的

可以使用*线程池*的方式来避免随连接建立断开带来的线程的频繁创建和销毁
线程池提前创建若干个线程，当新连接建立时，将已连接 Socket 放入到全局 socket 队列里，然后线程池里的线程加锁访问队列，取出已连接 Socket 进行处理

在连接数较多时多线程模型依然负担较大

### IO 多路复用

一个进程虽然任一时刻只能处理一个请求，但是处理每个请求的事件时，耗时控制在 1 毫秒以内，这样 1 秒内就可以处理上千个请求，把时间拉长来看，多个请求复用了一个进程，这就是*多路复用*

内核提供给用户态的多路复用系统调用方法包括 select/poll/epoll，**进程可以通过一个系统调用函数从内核中获取多个事件**，在获取事件时，先把所有连接(文件描述符)传给内核，再由内核返回产生了事件的连接，然后在用户态中再处理这些连接对应的请求

> [!Attention]
> **I/O 多路复用 API 返回的事件并不一定是可读写的**(例如当数据已经到达但经检查后发现有错误的校验和而被丢弃)，如果使用阻塞 I/O，那么在调用 read/write 时则会发生程序阻塞，因此**多路复用最好搭配非阻塞 I/O**，以便应对极少数的特殊情况

#### Select/Poll

Select 方法使用一个文件描述符集合存放所有已连接 socket
1. 调用 select 函数将文件描述符集合拷贝到内核里，让内核通过遍历文件描述符集合的方式检查是否有网络事件产生
2. 当检查到有事件产生后，将此 Socket 标记为可读或可写
3. 接着再把整个文件描述符集合拷贝回用户态里
4. 然后用户态再通过遍历的方法找到可读或可写的 socket，然后再对其处理

所以，对于 select 这种方式，需要进行 2 次遍历文件描述符集合，一次是在内核态里，一次是在用户态里 ，而且还会发生 2 次拷贝文件描述符集合，先从用户空间传入内核空间，由内核修改后，再传出到用户空间中

select 使用**固定长度的 BitsMap**表示文件描述符集合，而且所支持的文件描述符的个数是有限制的，Linux 系统中由内核中的 `FD_SETSIZE` 限制， 默认最大值为 1024，只能监听 0~1023 的文件描述符

Poll 用**动态数组**代替 BitsMap，以链表形式来组织文件描述符，文件描述符个数仅受系统最大限制

但是 poll 和 select 都使用*线性结构*存储进程关注的 Socket 集合，都**需要遍历文件描述符集合来找到可读或可写的 Socket**，时间复杂度为 O(n)，而且也需要在用户态与内核态之间拷贝文件描述符集合，这种方式随着并发数上来，性能的损耗会呈指数级增长

#### Epoll

```C
int s = socket(AF_INET, SOCK_STREAM, 0);
bind(s, ...);
listen(s, ...)

int epfd = epoll_create(...);
epoll_ctl(epfd, ...); // 将所有需要监听的socket添加到epfd中

while(1) {
    int n = epoll_wait(...); // 等待数据
    for(接收到数据的socket){
        //处理
    }
}
```

> ![epoll.png (1142×527) (xiaolincoding.com)|600](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8/epoll.png)

Epoll 在内核里**使用*红黑树*维护了单个进程所有待检测的文件描述符**，进程仅需通过 `epoll_ctl()` 将待检测 socket 传入内核一次，而不用每次将所有 socket 的集合传入
Epoll 使用**事件驱动机制**，内核里维护了一个链表来记录就绪事件，当某个 socket 有事件发生时，内核会通过回调函数将其加入到链表中，进程调用 `epoll_wait()` 函数时，只会返回有事件发生的文件描述符，不需要像 select/poll 那样轮询扫描整个 socket 集合，大大提高了检测的效率

> 红黑树增删改一般时间复杂度是 `O(logn)`

Epoll 方式下 socket 数据较多时不会对效率造成较大影响，同时文件描述符个数仅受系统最大限制，可以很好地解决 C10K 问题

> [!Note] C10K 问题
> C10K 问题指的是如何处理并发连接数超过10,000个的问题
> 解决 C10K 问题需要在保证可靠性和性能的前提下，合理地组织网络架构、调度线程和优化算法等方面进行技术探索。具体的解决思路包括：
> 异步/事件驱动编程模型：采用异步/事件驱动模型可以极大地提高并发处理能力，例如基于事件驱动的 Reactor 模式和基于异步回调的 Proactor 模式
> 多线程/多进程模型：利用多线程/多进程模型可以扩展处理能力，但需要解决线程/进程同步和资源管理的问题。常见的方法有分离逻辑线程和网络线程、使用线程池与任务队列管理等
> 集群/分布式系统：利用分布式架构的方式，可以将负载分散到不同的服务器上，以实现扩展性和容错性。常见的方法包括负载均衡、无状态架构、数据分片、副本与冗余备份等
> 内核/协议栈优化：针对网络协议栈进行优化，可以减少系统调用次数，降低内存复制和锁等带来的性能损耗，提升网络响应速度和吞吐量。

Epoll 支持*边缘触发*(ET)和*水平触发*(LT)两种事件触发模式
- ET 模式下，epoll 对被监控的文件描述符信息的变化只有在状态发生变化的那一刻才通知程序，程序需要一次性将数据全部读出，否则只读了一部分数据的文件描述符映射区域将一直保持就绪状态，此时 epoll 永远不会通知程序。因此 ET 模式下程序通常会循环从文件描述符读写数据，如果文件描述符是阻塞的，没有数据可读写时，进程会阻塞在读写函数那里，程序就没办法继续往下执行。所以**ET 模式一般和非阻塞 I/O 搭配使用**，程序会一直执行 I/O 操作，直到系统调用(如 read 和 write)返回错误，错误类型为 `EAGAIN` 或 `EWOULDBLOCK`
- LT 模式(default)下，在被监控的 Socket 描述符上有可读事件发生时 epoll 不断地通知程序，直到内核缓冲区数据被 read 函数读完才停止，如果应用程序在某一次通知时只读取部分数据，那么下一次仍然会通知它上一次已经读取的数据量，在 LT 模式下当内核通知文件描述符可读写时，接下来还可以继续去检测它的状态，看它是否依然可读或可写，所以在收到通知后，没必要一次执行尽可能多的读写操作

一般来说，ET 模式的效率比 LT 的效率要高，因为边缘触发可以减少 epoll_wait 的系统调用次数
Select/poll 只有水平触发模式，epoll 默认的触发模式是水平触发，但是可以根据应用场景设置为边缘触发模式

### 零拷贝

在 [[Code/4.OS/OS.0d.文件系统#DMA\|DMA技术]]下，文件传输仍然会发生多次上下文切换，可通过零拷贝技术减少切换提高效率
> ![传统文件传输.png (1100×678) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E9%9B%B6%E6%8B%B7%E8%B4%9D/%E4%BC%A0%E7%BB%9F%E6%96%87%E4%BB%B6%E4%BC%A0%E8%BE%93.png)

可通过 `mmap + write` 或 `sendfile` 方式实现零拷贝减少文件传输中的上下文切换

####  `mmap + write` 
通过使用 mmap()替换 read()函数，可直接将内核缓冲区里的数据映射到用户空间以避免数据在内核空间和用户空间之间的拷贝
> ![mmap + write 零拷贝.png (1100×677) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E9%9B%B6%E6%8B%B7%E8%B4%9D/mmap%20%2B%20write%20%E9%9B%B6%E6%8B%B7%E8%B4%9D.png)

####  `sendfile`

可以使用 sendfile 替代 read+write，直接将内核缓冲区中的数据拷贝到 socket 缓冲区中
> ![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E9%9B%B6%E6%8B%B7%E8%B4%9D/senfile-3%E6%AC%A1%E6%8B%B7%E8%B4%9D.png)

如果网卡支持 SG-DMA 技术，sendfile 将缓冲区描述符和数据长度传到 socket 缓冲区后，网卡的 SG-DMA 控制器就可以直接将内核缓存中的数据拷贝到网卡的缓冲区里

这种最终的**零拷贝**技术全程没有通过 CPU 来搬运数据，所有数据都通过 DMA 来进行传输

>![senfile-零拷贝.png (1160×686) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost2/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E9%9B%B6%E6%8B%B7%E8%B4%9D/senfile-%E9%9B%B6%E6%8B%B7%E8%B4%9D.png)

## 高性能网络模式

### Reactor

Reactor 模式也叫 Dispatcher 模式，即 **IO 多路复用监听事件，收到事件后，根据事件类型分配(Dispatch)给某个进程 / 线程**

Reactor 模式主要由 Reactor 和处理资源池两个核心部分组成
- Reactor 负责监听和分发事件，事件类型包含连接事件、读写事件；
- 处理资源池负责处理事件，如 read -> 业务逻辑 -> send；

#### 单 Reactor 单进程/线程

进程里有 **Reactor、Acceptor、Handler** 三个对象：
- Reactor 监听和分发事件
- Acceptor 获取连接
- Handler 处理业务

对象里的 select、accept、read、send 是系统调用函数，dispatch 和业务处理 是需要完成的操作，其中 dispatch 是分发事件操作

> ![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Reactor/%E5%8D%95Reactor%E5%8D%95%E8%BF%9B%E7%A8%8B.png)

Reactor 对象通过 select (IO 多路复用接口) 监听事件，收到事件后通过 dispatch 进行分发，根据收到事件的类型分别发送给 Acceptor 对象和 Handler 对象
- 如果是连接建立的事件，则交由 Acceptor 对象进行处理，Acceptor 对象会通过 accept 方法获取连接，并创建一个 Handler 对象来处理后续的响应事件
- 如果不是连接建立事件，则交由当前连接对应的 Handler 对象来进行响应，Handler 对象通过 read -> 业务处理 -> send 的流程来完成完整的业务流程

缺点：
- 无法充分利用多核 CPU 性能
- 业务处理阻塞整个进程

#### 单 Reactor 多线程

> ![单Reactor多线程.png (1514×1277) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Reactor/%E5%8D%95Reactor%E5%A4%9A%E7%BA%BF%E7%A8%8B.png)

相较于单 Reactor 单线程方案，多线程方案中 Handler 对象不再负责业务处理，只负责数据的接收和发送，Handler 对象通过 read 读取到数据后，会将数据发给子线程里的 Processor 对象进行业务处理，子线程里的 Processor 对象处理完后，将结果发给主线程中的 Handler 对象，接着由 Handler 通过 send 方法将响应结果发送给 client

问题：多线程竞争共享资源

#### 多 Reactor 多线程

> ![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Reactor/%E4%B8%BB%E4%BB%8EReactor%E5%A4%9A%E7%BA%BF%E7%A8%8B.png)

1. 主线程中的 MainReactor 对象通过 select 监控连接建立事件，收到事件后通过 Acceptor 对象中的 accept 获取连接，将新的连接分配给某个子线程
2. 子线程中的 SubReactor 对象将 MainReactor 对象分配的连接加入 select 继续进行监听，并创建一个 Handler 用于处理连接的响应事件
3. 新的事件发生时 SubReactor 对象调用当前连接对应的 Handler 对象进行响应
4. Handler 对象通过 read -> 业务处理 -> send 的流程来完成完整的业务流程。

优点：
- 主线程和子线程分工明确，主线程只负责接收新连接，子线程负责完成后续的业务处理
- 主线程和子线程的交互简单，主线程只需要把新连接传给子线程，子线程无须返回数据，直接就可以在子线程将处理结果发送给客户端

### Proactor

> [[Code/4.OS/OS.0d.文件系统#阻塞/非阻塞 IO&同步/异步 IO\|OS.0d.文件系统#阻塞/非阻塞 IO&同步/异步 IO]]
- Reactor 是**非阻塞同步网络模式**，感知的是**就绪可读写事件**
  - 在每次感知到有事件发生(比如可读就绪事件)后，Reactor 需要应用进程主动调用 read 方法来读取数据，读取过程中应用进程需主动将 socket 接收缓存中的数据按照同步方式读到应用进程内存中，等待读取完数据后应用进程才能处理数据
- Proactor 是**异步网络模式**，感知的是**已完成的读写事件**
  - 在发起异步读写请求时，需要传入数据缓冲区的地址(用来存放结果数据)等信息，系统内核自动完成数据的读写工作，将数据拷贝至应用进程数据缓冲区后通知应用进程处理数据

> ![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/Reactor/Proactor.png)

1. Proactor Initiator 负责创建 Proactor 和 Handler 对象，并将 Proactor 和 Handler 都通过 Asynchronous Operation Processor 注册到内核
2. Asynchronous Operation Processor 负责处理注册请求，并处理 IO 操作，完成 IO 操作后通知 Proactor
3. Proactor 根据不同的事件类型回调不同的 Handler 进行业务处理
4. Handler 完成业务处理

Linux 下异步 IO 不完善，基于 Linux 的高性能网络程序都是使用 Reactor 方案
Windows 实现了一套完整的支持 socket 的异步编程接口 `IOCP`，在 Windows 里实现高性能网络程序可以使用效率更高的 Proactor 方案

## 一致性 Hash

一致性 Hash 用于解决分布式系统的负载均衡问题

对于分布式系统，每个节点存储的数据不同，每个 key 的获取节点是固定的

hash 算法使用数据的 hash 值对节点数取余来确定数据所在的节点
缺点是在节点数量变化时会产生数据的大量迁移

一致性 Hash 算法将存储节点和数据都通过对 2^32 进行取模运算映射到一个首尾相接的哈希环上

将数据映射的结果值向顺时针方向找到的第一个节点即为存储该数据的节点

> ![30c2c70721c12f9c140358fbdc5f2282.png (587×560) (xiaolincoding.com)](https://cdn.xiaolincoding.com//mysql/other/30c2c70721c12f9c140358fbdc5f2282.png)

一致性 hash 算法的缺点在于**不保证节点能够在哈希环上分布均匀**，因此可能会有大量请求集中在一个节点上

解决方式是增加虚拟节点，将一个真实节点映射至多个虚拟节点，再将虚拟节点映射至 hash 环

当节点数据较多时，节点在 hash 环上的分布相对均匀
