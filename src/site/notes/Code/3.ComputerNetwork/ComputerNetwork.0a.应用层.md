---
{"dg-publish":true,"permalink":"/code/3-computer-network/computer-network-0a/","title":"应用层"}
---


# 应用层

网络应用程序仅运行在端系统上，不涉及[[Code/3.ComputerNetwork/ComputerNetwork.0.计算机网络#^internetCore\|网络核心]]

**应用程序体系结构** (application architecture)：
- 客户-服务器(C/S)
	- 服务器存储资源，客户端仅向服务器请求资源，
	- 服务器持续运行，具有固定的IP地址和周知的端口号
	- 可扩展性差
- 对等模式P2P(peer-to-peer)
	- 任意端系统都能相互通信
	- 自扩展性(self-scalability)
	- 管理困难
- 混合模式
	- 客户端登录时向服务器注册
	- 端系统相互通信

## 应用层通信原理

网络应用依赖于进程端口的相互通信，在同一个主机内使用进程间通信机制通信，在不同主机上通过交换报文来通信
- 进程：在主机上运行的应用程序
- 客户端进程：发起通信的进程
- 服务器进程：等待通信的进程
- P2P中也存在客户端进程和服务器进程

应用层通过IP地址和端口号确定传输双方的身份
每个主机具有唯一的一个32位的IP地址
每个进程具有一个**端口号**(port number)，流行的应用协议分配有特定的端口号
TCP、UDP 分别实现了端口，因此两个协议的端口不会互相影响。

**应用层协议**规定了不同端系统上的应用进程如何相互交换报文
- 交换的报文类型，例如请求报文和响应报文。
- 各种报文类型的语法，如报文中的各个字段及这些字段是如何描述的。
- 字段的语义，即这些字段中的信息的含义。
- 确定一个进程何时以及如何发送报文，对报文进行响应的规则。

## 应用层相关协议

### HTTP协议

**HTTP**(HyperText Transfer Protocol，超文本传输协议(*严谨译名应为超文本转移协议*))是Web文档传递的规范


<div class="transclusion internal-embed is-loaded"><a class="markdown-embed-link" href="/code/3-computer-network/computer-network-1-http/#http" aria-label="Open link"><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="svg-icon lucide-link"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"></path><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"></path></svg></a><div class="markdown-embed">



## HTTP基础知识

HTTP协议用于客户端和服务端的通信，使用 **URI** 定位互联网上的资源，通过**请求报文**和**响应报文**建立通信

HTTP协议基于TCP协议，默认应用端口为80
> [!attention] 
> HTTP/3改用UDP协议

### HTTP报文

#### 报文内容

请求报文是由**请求方法**、**请求 URI**、**协议版本**、**可选的请求首部字段**和**内容实体**构成的，编码格式为ASCII

![|600](https://image.jiang849725768.asia/2022/202211202049360.png)

响应报文基本上由**协议版本**、**状态码**(表示请求成功或失败的数字代码)、**用以解释状态码的原因短语**、**可选的响应首部字段**以及**实体主体**构成。

![|600](https://image.jiang849725768.asia/2022/202211202054456.png)

请求报文：
- 报文首部
	- 请求行 - 用于请求的方法，请求 URI 和 HTTP 版本
	- 请求首部字段 - 从客户端向服务器端发送请求报文时使用的首部。补充了请求的附加内容、客户端信息、响应内容相关优先级等信息。
	- 通用首部字段 - 请求报文和响应报文两方都会使用的首部。
	- 实体首部字段 - 针对请求报文和响应报文的实体部分使用的首部。补充了资源内容更新时间等与实体有关的信息。
	- 其他 - 包含 HTTP 的 RFC 里未定义的其他首部(Cookie 等)
- 空行(CR+LF) - CR：回车符；LF：换行符
- 报文主体(*非必需*)

响应报文：
- 报文首部
	- 状态行 - HTTP 版本，表明响应结果的状态码和原因短语
	- 响应首部字段 - 从服务器端向客户端返回响应报文时使用的首部。补充了响应的附加内容，也会要求客户端附加额外的内容信息。
	- 通用首部字段
	- 实体首部字段
	- 其他
- 空行(CR+LF)
- 报文主体(*非必需*)

**报文(message)**：HTTP 通信中的基本单位，由**8位组字节流**(octet sequence，其中 octet 为 8 个比特)组成，通过 HTTP 通信传输。
**实体(entity)**：作为请求或响应的有效载荷数据(补充项)被传输，其内容由**实体首部**和**实体主体**组成。

*通常，报文主体等于实体主体*。传输中进行编码操作时实体主体的内容发生变化。

##### 首部字段

HTTP首部字段格式为`{首部字段名}：{字段值}`
首部字段用于给浏览器和服务器提供报文主体大小、所使用的语言、认证信息等内容。
对单一报文中相同首部字段名重复出现的处理在规范内尚未明确，依赖于浏览器的内部处理逻辑
标准中没有对每个协议头字段的名称和值的大小设置任何限制，也没有限制字段的个数。然而，出于实际场景及安全性的考虑，*大部分的服务器、客户端和代理软件都会实施一些限制*。

HTTP 首部字段将定义成缓存代理和非缓存代理的行为，分成 2 种类型：
- **端到端首部**(End-to-end Header)
	- 此类别中的首部会转发给请求 / 响应对应的最终接收目标，且必须保存在由缓存生成的响应中，另外规定它必须被转发。
- **逐跳首部**(Hop-by-hop Header)
	- 此类别中的首部只对单次转发有效，会因通过缓存或代理而不再转发。*HTTP/1.1 和之后版本中，如果要使用 hop-by-hop 首部，需提供 Connection 首部字段。*
	- HTTP/1.1中的逐跳首部字段：Connection、Keep-Alive、Proxy-Authenticate、Proxy-Authorization、Trailer、TE、Transfer-Encoding、Upgrade

具体的首部字段内容及说明可查阅[HTTP头字段、-、维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/HTTP%E5%A4%B4%E5%AD%97%E6%AE%B5#%E5%B8%B8%E8%A7%81%E7%9A%84%E9%9D%9E%E6%A0%87%E5%87%86%E5%9B%9E%E5%BA%94%E5%AD%97%E6%AE%B5)

请求报文首部示例：

```http
GET / HTTP/1.1
Host: hackr.jp
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64; rv:13.0) Gecko/20100101 Firefox/13.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*; q=0.8
Accept-Language: ja,en-us;q=0.7,en;q=0.3
Accept-Encoding: gzip, deflate
DNT: 1
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache

```

响应报文首部示例：

```http
HTTP/1.1 200 OK
Date: Fri,13 Jul 2012 02:45:26 GMT
Server: Apache
Last-Modified: Fri,31 Aug 2007 02:02:20 GMT
BTag: "45bae1-16a-46d776ac"
Accept-Ranges: bytes
Content-Length: 362
Connection: close
Content-Type: text/html

```

##### HTTP请求方法

常见HTTP方法包括：

|  方法   |          作用          |   协议   |
|:-------:|:----------------------:|:--------:|
|   GET   |        获取资源        | 1.0、1.1 |
|  POST   |      传输实体主体      | 1.0、1.1 |
|  HEAD   |    获取报文首部字段    | 1.0、1.1 |
|   PUT   |        传输文件        | 1.0、1.1 |
| DELETE  |        删除文件        | 1.0、1.1 |
| OPTIONS |      询问支持方法      |   1.1    |
|  TRACE  |        追踪路径        |   1.1    |
| CONNECT | 要求用隧道协议连接代理 |   1.1    |
|  PATCH  |   对资源应用部分修改   |   1.1    |

##### HTTP状态码

|     |              类别               |          原因短语          |
|:---:|:-------------------------------:|:--------------------------:|
| 1XX |  Informational (信息性状态码)   |     接收的请求正在处理     |
| 2XX |      Success (成功状态码)       |      请求正常处理完毕      |
| 3XX |   Redirection (重定向状态码)    | 需要进行附加操作以完成请求 |
| 4XX | Client Error (客户端错误状态码) |     服务器无法处理请求     |
| 5XX | Server Error (服务器错误状态码) |     服务器处理请求出错     |

具体代码及含义可直接查阅[HTTP 状态代码概述 - Internet Information Services | Microsoft Learn](https://learn.microsoft.com/zh-cn/troubleshoot/developer/webapps/iis/www-administration-management/http-status-code)

### 数据传输

需传输的数据编码压缩后，分块传输，由客户端进行解码恢复
- 常见内容编码包括：
	- **gzip**(GNU zip)
	- **compress**(UNIX 系统的标准压缩)
	- **deflate**(zlib)
	- **identity**(不进行编码)
- 块使用十六进制标记大小，实体主体的最后一块会使用“0(CR+LF)”来标记

对于包含多种数据类型的实体，采用**多部分对象集合**并在首部字段中加入`Content-type`
- 多部分对象集合的每个部分类型中，都可以含有首部字段。另外，可以在某个部分中嵌套使用多部分对象集合。

对于断连后的重新传输，可以通过**范围请求**指定下载的实体范围，仅请求部分资源，请求报文首部字段包含`Range`

HTTP/1.1 和部分HTTP/1.0使用**持久连接**，客户端和服务端任意一端提出断开前保持TCP连接，HTTP/1.1中所有链接默认为持久连接
HTTP/1.1中使用持久连接使请求以**管线化**(pipelining)方式发送，实现同时并行发送多个请求，新请求的发送不需要等待上一个请求对应响应的接收

#### 状态管理

**HTTP协议为无状态协议**，不对之前发生过的请求和响应的状态进行管理，需使用**Cookies**记录状态
客户端和服务端**首次通信**时，服务器端通过响应报文内的`Set-Cookie`的首部字段信息通知客户端保存服务器发送的 Cookie，下次客户端向该服务器发送请求时自动在请求报文中加入 Cookie 值后发送
服务端检查Cookie获取状态信息

### 返回内容

服务器存在多个相同内容的页面时，通过内容协商机制返回合适内容
**内容协商机制**指客户端和服务器端就响应的资源内容进行交涉，然后提供给客户端最为适合的资源。
内容协商以响应资源的*语言、字符集、编码方式*等作为判断的基准。

机制类型：
- 服务器驱动协商(Server-driven Negotiation)
	- 由服务器端进行内容协商。以请求的首部字段为参考，在服务器端自动处理。
- 透明协商(Transparent Negotiation)
	- 是服务器驱动和客户端驱动的结合体，是由服务器端和客户端各自进行内容协商的一种方法。
- 客户端驱动协商(Agent-driven Negotiation)
	- 由客户端进行内容协商的方式。用户从浏览器显示的可选项列表中手动选择。


</div></div>


同时HTTP通过代理服务器减轻网络负载

<div class="transclusion internal-embed is-loaded"><a class="markdown-embed-link" href="/code/3-computer-network/computer-network-1-http/#" aria-label="Open link"><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="svg-icon lucide-link"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"></path><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"></path></svg></a><div class="markdown-embed">



#### 代理/缓存

**代理服务器**(proxy server)也叫**Web缓存器**(Web cache)，其接收客户端发送的请求后转发给其他服务器。代理*不改变请求 URI*，会直接发送给前方持有资源的目标服务器。
每次通过代理服务器转发请求或响应时，会追加写入`Via`首部信息

*用途*：利用缓存技术减少网络带宽的流量，组织内部针对特定网站的访问控制，以获取访问日志为主要目的等等

*类型*：**缓存/非缓存代理**、**透明/非透明代理**
代理转发响应时，缓存代理(Caching Proxy)会预先将资源的副本(缓存)保存在代理服务器上
转发请求或响应时，不对报文做任何加工的代理类型被称为透明代理

**缓存服务器**是代理服务器的一种，并归类在缓存代理类型中，缓存可以存在于缓存服务器内或客户端浏览器中。判定缓存过期后，会向源服务器确认资源的有效性，若判断浏览器缓存失效，浏览器会再次请求新资源。***缓存服务器既是服务器又是客户端。*** 当它接收浏览器的请求并发回响应时是服务器。当它向初始服务器发出请求并接收响应时是客户端。

- 降低客户端响应时间
- 减轻服务器负载

对于缓存的版本问题，HTTP使用**条件GET**(conditional GET)方法，在GET方法的首部字段中加入`If-Modified-Since`字段以确认缓存文件是否改变


</div></div>


### FTP

**文件传输协议**(File Transfer Protocol，FTP)是一个用于在计算机网络上在客户端和服务器之间进行文件传输的应用层协议。默认控制端口号为21，传输端口号为20

FTP用于向远程主机上传输文件或从远程主机接收文件，使用客户-服务器结构

与HTTP相比，FTP维护用户状态，同时控制与数据传输采用不同的连接，通过控制连接发送命令及响应，使用数据传输连接来传输文件
在控制连接上以*ASCII文本*方式传送命令及响应

目前为止(2022-12-06)，FTP协议在主流浏览器中基本被弃用

### EMail

互联网电子邮件系统由**用户代理**(user agent)、**邮件服务器**(mail server) 和**简单邮件传输协议**(Simple Mail Transfer Protocol, **SMTP**)三部分组成
SMTP使用客户-服务器结构，默认端口为25，报文必须为7位ASCII码

用户代理为Outlook、Gmail等邮件处理应用
每个用户在邮件服务器上有一个邮箱，邮箱管理和维护发送给该用户的邮件

邮件：发送方用户代理->发送方服务器->报文队列->SMTP握手->基于SMTP发送email报文->接收方服务器->接收方邮箱->POP3/IMAP/HTTP->接收方用户代理

**POP**：邮局访问协议(Post Office Protocol)
- 无状态
- 用户将邮件下载至本地管理
**IMAP**：Internet邮件访问协议(Internet Mail Access Protocol)
- 会话过程中保留用户状态
- 将报文与文件夹关联，用户远程管理文件夹

发送失败：发送方用户代理->发送方服务器->报文队列->发送失败->间隔一段时间再次发送->多次失败->服务器删除报文并以email通知发送方

SMTP与HTTP的区别：
- HTTP为拉协议(pull protocol)，客户端向服务器请求拉取文件，SMTP为推协议(push protocol)
- HTTP报文不受ASCII码限制
- HTTP把每个对象封装到对象自身的HTTP响应报文中，SMTP则把所有报文对象放在一个报文之中

邮件报文格式：
![|525](https://image.jiang849725768.asia/2022/202212072023591.png)

**多媒体邮件扩展MIME**(multimedia mail extension)用于报文扩展，使用指定编码方式将非ASCII码内容编码为ASCII码发送

### DNS

**域名系统DNS**(Domain Name System)提供域名到IP地址的转换

DNS是：
- 一个由分层的DNS服务器(DNS server) 实现的分布式数据库
- 一个主机用于查询分布式数据库的应用层协议。
DNS协议运行在UDP之上，使用客户-服务器结构，默认端口为53

DNS功能：
- 主机名-IP地址的转换
- 主机别名( host aliasing) - 规范主机名(canonical hostname) 的转换
	- 有着复杂主机名的主机拥有一个或者多个别名
- 邮件服务器别名-规范主机名的转换
- 负载分配( load distribution)
	- 繁忙站点被冗余分布在多台服务器上，其规范主机名连接着一个IP地址集合

>[!note] 域名
>主机域名往往采用层次树状结构的命名方法，域名命名从本域往上，直到树根，中间使用“.”间隔不同的级别
>例如：`ustc.edu.cn`、`auto.ustc.edu.cn`、`www.auto.ustc.edu.cn`
>Internet 根被划为几百个顶级域(top lever domains)，每个(子)域下面可划分为若干子域(subdomains)，每个域管理自己的子域
>域的域名：可以用于表示一个域
>主机的域名：一个域上的一个主机

DNS服务器类型：
- 根DNS服务器
	- 提供TLD服务器的IP地址
- 顶级域(Top-Level Domain, TLD) DNS服务器
	- 负责顶级域名(如com, org, net,edu和gov)和所有国家级的顶级域名(如cn, uk, fr, ca,jp )
	- 提供权威DNS服务器的IP地址
- 权威DNS服务器
	- 组织机构的DNS服务器， 提供组织机构服务器(如Web和mail)可访问的主机和IP之间的映射
- 本地DNS服务器(Local DNS Server)
	- 严格来说不属于DNS系统的层次结构
	- 每个ISP都有一台本地DNS服务器

DNS工作过程：
- 递归查询![|375](https://image.jiang849725768.asia/2022/202212081811082.png)
	- 根节点负担重
- 迭代查询![|325](https://image.jiang849725768.asia/2022/202212081814132.png)
	- 根(及各级域名)服务器返回的不是查询结果，而是下一个NS的地址
	- 通过缓存可减少对根节点的请求

>[!note] DNS缓存
>在一个请求链中，当某DNS服务器接收到一个DNS回答(例如某主机名到IP地址的映射)时将映射缓存在本地存储器中，并在一定时间后丢弃。

DNS服务器将主机名到IP地址的映射关系以**资源记录**(Resource Record，RR)的形式进行存储
每条记录都包含`(Domain_Name, TTL, Type, Class, Value)`
- Domain_name: 域名
- TTL: time to live, 生存时间
- Class 类别：对于Internet，值为IN(可维护非Internet值)
- Value 值：可以是数字，域名或ASCII串
- Type 类别：资源记录的类型
	- A - Name=主机，Value=IP地址
	- CNAME - Name=别名，Value=规范主机名
	- NS - Name=域名，Value=该域名所属的权威服务器的域名
	- MX -Name=邮件服务器别名，Value=规范主机名

DNS协议：查询和响应报文的报文格式相同
报文首部包含12个字节，又分为六个字段，第一个字段为报文标识符(id)，第二个字段为标志，包含对报文类型的各种标志，如1bit的“查询/回答”标志位指出报文是查询报文(0)还是回答报文(1)

## P2P

纯P2P架构：
- 基本没有一直运行的服务器
- 任意端系统可以直接通信
- peer节点每次连接的IP地址可能变化

非结构化P2P：节点间随机连接
- 集中式目录，中心服务器管理资源关系
- 完全分布式：泛洪式查询，e.g. Gnutella
- 混合式：对等方连接组长，组长相互连接并管理资源关系，e.g. KaZaA
DHT(Distributed Hash Table)P2P(结构化)：节点间有序连接

通常使用Hash值作为文件的唯一标识

### BitTorrent

BitTorrent是一种用于文件分发的流行P2P协议

参与一个特定文件分发的所有对等方(peer)的集合被称为一个**洪流**(torrent) ，每个洪流具有一个基础设施节点**追踪器**(tracker) ，在一个洪流中的对等方彼此下载等长度的文件块(chunk)(通常大小为256KB)，一个对等方拥有块的情况由BitMap描述

当一个对等方X加入某洪流时
1. 它向追踪器注册自己并周期性地通知追踪器它仍在该洪流中
2. 追踪器随机地从参与对等方的集合中选择一个子集并将其中对等方的IP地址列表发送给X
3. X尝试与列表中的所有对等方建立并行的TCP连接，成功建立连接的对等方称为**邻近对等方**
4. X周期性询问每个邻近对等方所具有的块列表，并对其当前没有的块发出请求
5. 随时间流逝对等方可能会上线或者下线，邻近对等方会发生变化

请求块算法：**最稀缺优先**(rarest first)
针对当前没有的块在邻近对等方集合中的副本数量决定最稀缺的块，并首先请求那些最稀缺的块
发送块算法：**一报还一报**(tit-for-tat)
对每个邻近对等方实时监测块接收速率，向接收块速率最快的 4 个对等方优先发送块，同时每过 30 秒随机选择一个邻近对等方发送块

## CDN

### 互联网视频

视频流量占据着互联网大部分的带宽
视频是以一种恒定速率展现的图像，一幅未压缩、数字编码的图像由像素阵列组成，每个像素是由一些比特编码来表示亮度和颜色。
通过使用图像内和图像间的冗余来降低编码的比特数，视频能够被压缩，因而可用比特率(bps)来权衡视频质量。

用户观看流式视频时向服务器请求视频文件，接收的数据收集在应用缓存中，达到一定限度后应用程序开始播放视频，应用程序周期性地从缓存中抓取帧，对这些帧解压缩并且在屏幕上展现。因此，流式视频应用接收到视频就进行播放，同时缓存该视频后面部分的帧。

经 HTTP 的动态适应性流**DASH**(Dynamic Adaptive Streaming over HTTP) 将视频以不同比特率编码为几个不同的版本。客户动态地请求来自不同版本且长度为几秒的视频段数据块，根据实时带宽切换选择合适速率版本的块。
服务器通过一个**告示文件**(manifest file)为每个版本提供了一个URL及其比特率。客户端首先请求该告示文件，然后周期性地测量服务器到客户端的带宽，自适应决定请求块的时间，类型和位置，通过在HTTP GET请求报文中指定一个URL和一个字节范围获取视频的不同块。

### 内容分发网络CDN

CDN(Content Distribution Networks/Content Delivery Network)全网部署缓存服务器节点，存储服务内容，就近为用户提供服务

缓存服务器安置原则：
- enter deep：将CDN服务器部署至接近本地ISP的深入位置
- bring home：将服务器集中部署在少数关键位置，通常放置在IXP

CND在缓存服务器节点中存储内容的多个拷贝，用户请求内容时重定向到其中一个，从其中获取内容。
大多数CDN利用DNS来截获并重定向请求

## 套接字编程

应用进程使用传输层提供的服务交换报文，实现应用协议，进而实现应用

### 传输层提供的服务

应用层需要向下层传输层传输的内容：
- 报文
- 目标进程的标识
- 自身源进程的标识

传输层向应用层提供的接口为**套接字(socket)**
socket是一个用于标识并代表通信进程的仅具有本地意义的整数，作用是减少层间通信量
[[Code/3.ComputerNetwork/ComputerNetwork.0b.传输层#TCP\|TCP]]中socket标识4元组：(源IP，源port，目标IP，目标port)
[[Code/3.ComputerNetwork/ComputerNetwork.0b.传输层#UDP\|UDP]]中socket标识2元组：(本IP，本端口号)

传输能力的衡量标准：**数据丢失率**，**吞吐量**，**延迟**，**安全性**

### TCP套接字编程

1. S - 创建`welcome_socket`
2. S - `welcome_socket`显式绑定服务器ip与服务器指定端口sa[^1]
3. S - 阻塞，等待客户端连接
4. C - 创建`client_socket`
5. C - `client_socket`隐式绑定客户端ip与客户端自动分配端口ca
6. C - `client_socket`指定服务器主机和端口sa进行连接
7. S - 解除阻塞，获得客户端ip及端口ca，生成`connect_socket`
8. C - 发送请求报文(编码)
9. S - 接收请求报文(解码)，返回响应报文(编码)
10. C - 接收响应报文(解码)
11. C - 关闭`client_socket`

服务端：
```python
from socket import *

server_name = '127.0.0.1'
server_port = 12000
server_socket = socket(AF_INET,SOCK_STREAM)
server_socket.bind((server_name,server_port))
server_socket.listen(1)
print("The server is ready to receive")

while True:
    connection_socket, addr = server_socket.accept()
    print("Connect success!")
    message = connection_socket.recv(1024)
    print(message.decode())
    modified_message = message.decode().upper()
    connection_socket.send(modified_message.encode())
    connection_socket.close()

```
客户端：
```python
from socket import *

server_name = '127.0.0.1'
server_port = 12000
client_socket = socket(AF_INET,SOCK_STREAM)

client_socket.connect((server_name,server_port))
message = input("Input lowercase sentence\n")
client_socket.send(message.encode())
modified_message = client_socket.recv(1024)
print(modified_message.decode())
client_socket.close()
```

### UDP套接字编程

1. S - 创建`server_socket`
2. S - `server_socket`显式绑定服务器ip与服务器指定端口sa
3. S - 等待客户端连接
4. C - 创建`client_socket`
5. C - `client_socket`隐式绑定客户端ip与客户端自动分配端口ca
6. C - `client_socket`指定服务器主机和端口sa发送请求报文(编码)
9. S - 接收请求报文(解码)，获得客户端ip及端口ca，返回响应报文(编码)
10. C - 接收响应报文(解码)
11. C - 关闭`client_socket`

服务端：
```python
from socket import *

server_host = '127.0.0.1'
server_port = 12000
server_socket = socket(AF_INET, SOCK_DGRAM)
server_socket.bind((server_host, server_port))
print("The server is ready to receive")
while True:
    message, client_address = server_socket.recvfrom(2048)
    print(message.decode())
    modified_message = message.decode().upper()
    server_socket.sendto(modified_message.encode(), client_address)
```
客户端：
```python
from socket import *

server_name = '127.0.0.1'
server_port = 12000
client_socket = socket(AF_INET, SOCK_DGRAM)
message = input('Input lowercase sentence\n')

client_socket.sendto(message.encode(),(server_name,server_port))
print("Send success!")
modified_message, server_address = client_socket.recvfrom(2048)
print(modified_message.decode())
client_socket.close()
```

[^1]: sa,ca实际为一整数
