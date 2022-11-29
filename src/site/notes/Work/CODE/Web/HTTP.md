---
{"dg-publish":true,"dg-home":false,"tags":[],"categories":[],"description":null,"summary":null,"draft":true,"isCJKLanguage":true,"title":"HTTP","date":"2022-11-15","lastmod":"2022-11-28","permalink":"/work/code/web/http/","dgPassFrontmatter":true}
---


# HTTP

《图解HTTP》阅读笔记

## 前置知识

### 相关协议

**HTTP**（HyperText Transfer Protocol，超文本传输协议（严谨译名应为超文本转移协议））是web文档传递的规范，**WWW**（万维网）的三项构建技术之一，另两个是把 SGML（Standard Generalized Markup Language，标准通用标记语言）作为页面的文本标记语言的 **HTML**（HyperText Markup Language，超文本标记语言）以及指定文档所在地址的 **URL**（Uniform Resource Locator，统一资源定位符）

**DNS 协议**提供通过域名查找 IP 地址，或逆向从 IP 地址反查域名的服务

HTTP位于[[Work/CODE/Web/TCP-IP\|TCP/IP]]分层中的**应用层**，数据传输过程如下：
![](https://image.jiang849725768.asia/2022/202211261751316.png)

这种把数据信息包装起来的做法称为**封装**（encapsulate）

HTTP与各种协议的关系如下：
![|500](https://image.jiang849725768.asia/2022/202211212023993.png)

**URI**（Uniform Resource Identifier，统一资源定位标识符） 用字符串标识某一互联网资源，而 URL 表示资源的地点（互联网上所处的位置），后者是前者的子集
表示指定的 URI，要使用涵盖全部必要信息的绝对 URI、绝对 URL 以及相对 URL。

`http://user:pass@www.example.jp:80/dir/index.htm?uid=1#ch1`为**绝对URI**，不区分大小写
- `http`或`https`为协议方案名
- `user:pass`为登录信息（可选项）
	指定用户名和密码作为从服务器端获取资源时必要的登录信息
- `www.example.jp`为服务器地址
	类似这种 DNS 可解析的名称，或是 192.168.1.1 这类 IPv4 地址名，还可以是\[0:0:0:0:0:0:0:1\] 这样用方括号括起来的 IPv6 地址名
- `80`为服务器端口号（可选项）
	省略则自动使用默认端口号
- `/dir/index.htm`为带层次的文件路径
	指定服务器上的文件路径来定位特指的资源
- `uid=1`为查询字符串（可选项）
- `ch1`为片段标识符（可选项）

相对URL与带层次的文件路径类似

用来制定HTTP协议技术标准的文档被称为**RFC**（Request for Comments，征求修正意见书）

### Web构建

#### HTML

**HTML**（HyperText Markup Language，超文本标记语言）是为了发送Web 上的**超文本**（Hypertext）而开发的标记语言。
超文本是一种文档系统，可将文档中任意位置的信息与其他信息（文本或图片等）建立关联，即超链接文本。
标记语言是指通过在文档的某部分穿插特别的字符串标签，用来修饰文档的语言。
我们把出现在 HTML 文档内的这种特殊字符串叫做 **HTML 标签**（Tag）。

示例;
```html
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<title>hackr.jp</title>
<style type="text/css">
.logo {
padding: 20px;
text-align: center;
}
</style>
</head>
<body>
<div class="logo">
<p><img src="photo.jpg" alt="photo" width="240" height="127" /></p>
<p><img src="hackr.gif" alt="hackr.jp" width="240" height="84" /></p>
<p><a href="http://hackr.jp/">hackr.jp</a> </p>
</div>
</body>
</html>
```

HTML5是HTML最新的修订版本，广义论及HTML5时，实际指的是包括HTML、CSS和JavaScript在内的一套技术组合。

##### 动态HTML

**动态 HTML**（Dynamic HTML），是指使用客户端脚本语言将静态的 HTML 内容变成动态的技术的总称。
动态 HTML 技术是通过调用客户端脚本语言 **JavaScript**，实现对HTML 的 Web 页面的动态改造。
利用 **DOM**（Document ObjectModel，文档对象模型）可指定欲发生动态变化的 HTML 元素。

#### CSS

CSS（Cascading Style Sheets，层叠样式表）可以指定如何展现 HTML内的各种元素，属于样式表标准之一。
即使是相同的 HTML 文档，通过改变应用的 CSS，用浏览器看到的页面外观也会随之改变。

```css
.logo {
padding: 20px;
text-align: center;
}
```

#### Web应用

由程序创建的HTML内容称为**动态内容**，而事先准备好的HTML内容称为**静态内容**。

**CGI**（Common Gateway Interface，通用网关接口）是指 Web 服务器在接收到客户端发送过来的请求后转发给程序的一组机制。在 CGI 的作用下，程序会对请求内容做出相应的动作，比如创建 HTML 等动态内容。

**Servlet** 是一种能在服务器上创建动态内容的程序。Servlet 是用 Java语言实现的一个接口，属于面向企业级 Java（JavaEE，JavaEnterprise Edition）的一部分。

#### XML

**XML**（eXtensible Markup Language，可扩展标记语言）是一种可按应用目标进行扩展的通用标记语言。
XML 和 HTML 都是从标准通用标记语言 **SGML**（Standard GeneralizedMarkup Language）简化而成。与 HTML 相比，它对数据的记录方式做了特殊处理。

#### JSON

**JSON**（JavaScript Object Notation）是一种以JavaScript（ECMAScript）的对象表示法为基础的轻量级数据标记语言。能够处理的数据类型有 *布尔值/空值/对象/数组/数字/字符串*，这7种类型。

## HTTP基础知识

HTTP协议用于客户端和服务端的通信，使用 **URI** 定位互联网上的资源，通过**请求报文**和**响应报文**建立通信

### HTTP报文

#### 报文内容

请求报文是由**请求方法**、**请求 URI**、**协议版本**、**可选的请求首部字段**和**内容实体**构成的

![|600](https://image.jiang849725768.asia/2022/202211202049360.png)

响应报文基本上由**协议版本**、**状态码**（表示请求成功或失败的数字代码）、**用以解释状态码的原因短语**、**可选的响应首部字段**以及**实体主体**构成。

![|600](https://image.jiang849725768.asia/2022/202211202054456.png)

请求报文：
- 报文首部
	- 请求行 - 用于请求的方法，请求 URI 和 HTTP 版本
	- 请求首部字段 - 从客户端向服务器端发送请求报文时使用的首部。补充了请求的附加内容、客户端信息、响应内容相关优先级等信息。
	- 通用首部字段 - 请求报文和响应报文两方都会使用的首部。
	- 实体首部字段 - 针对请求报文和响应报文的实体部分使用的首部。补充了资源内容更新时间等与实体有关的信息。
	- 其他 - 可能包含 HTTP 的 RFC 里未定义的首部（Cookie 等）
- 空行（CR+LF) - CR：回车符；LF：换行符
- 报文主体(**非必需**)

响应报文：
- 报文首部
	- 状态行 - HTTP 版本，表明响应结果的状态码和原因短语
	- 响应首部字段 - 从服务器端向客户端返回响应报文时使用的首部。补充了响应的附加内容，也会要求客户端附加额外的内容信息。
	- 通用首部字段
	- 实体首部字段
	- 其他
- 空行（CR+LF)
- 报文主体(**非必需**)

**报文（message）**：
HTTP 通信中的基本单位，由**8位组字节流**（octet sequence，其中 octet 为 8 个比特）组成，通过 HTTP 通信传输。
**实体（entity）**：
作为请求或响应的有效载荷数据（补充项）被传输，其内容由**实体首部**和**实体主体**组成。

*通常，报文主体等于实体主体*。只有当传输中进行编码操作时，实体主体的内容发生变化，才导致它和报文主体产生差异。

##### 首部字段

HTTP首部字段格式为`{首部字段名}：{字段值}`
使用首部字段是为了给浏览器和服务器提供报文主体大小、所使用的语言、认证信息等内容。
对单一报文中相同首部字段名重复出现的处理在规范内尚未明确，依赖于浏览器的内部处理逻辑
标准中没有对每个协议头字段的名称和值的大小设置任何限制，也没有限制字段的个数。然而，出于实际场景及安全性的考虑，*大部分的服务器、客户端和代理软件都会实施一些限制*。

HTTP 首部字段将定义成缓存代理和非缓存代理的行为，分成 2 种类型：
- **端到端首部**（End-to-end Header）
	- 分在此类别中的首部会转发给请求 / 响应对应的最终接收目标，且必须保存在由缓存生成的响应中，另外规定它必须被转发。
- **逐跳首部**（Hop-by-hop Header）
	- 分在此类别中的首部只对单次转发有效，会因通过缓存或代理而不再转发。*HTTP/1.1 和之后版本中，如果要使用 hop-by-hop 首部，需提供 Connection 首部字段。*
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
- 常见内容编码包括：**gzip**（GNU zip） **compress**（UNIX 系统的标准压缩） **deflate**（zlib） **identity**（不进行编码）
- 块会用十六进制来标记大小，实体主体的最后一块会使用“0(CR+LF)”来标记

对于包含多种数据类型的实体，采用**多部分对象集合**并在首部字段中加入`Content-type`
- 多部分对象集合的每个部分类型中，都可以含有首部字段。另外，可以在某个部分中嵌套使用多部分对象集合。

对于断连后的重新传输，可以通过**范围请求**指定下载的实体范围，仅请求部分资源，请求报文首部字段包含`Range`

HTTP/1.1 和部分HTTP/1.0使用**持久连接**，客户端和服务端任意一端提出断开前保持TCP连接，HTTP/1.1中所有链接默认为持久连接
HTTP/1.1中使用持久连接使请求以**管线化**（pipelining）方式发送，实现同时并行发送多个请求，新请求的发送不需要等待上一个请求对应响应的接收

#### 状态管理

HTTP协议为无状态协议，不对之前发生过的请求和响应的状态进行管理，需使用**Cookies**记录状态
客户端和服务端*首次通信*时服务器端发送的响应报文内的`Set-Cookie`的首部字段信息通知客户端保存 Cookie，下次客户端往该服务器发送请求时自动在请求报文中加入 Cookie 值后发送
服务端检查Cookie获取状态信息

### 返回内容

服务器存在多个相同内容的页面时，通过**内容协商**机制返回合适内容
内容协商机制指客户端和服务器端就响应的资源内容进行交涉，然后提供给客户端最为适合的资源。内容协商会以响应资源的*语言、字符集、编码方式*等作为判断的基准。

- 机制类型
	- 服务器驱动协商（Server-driven Negotiation）
		由服务器端进行内容协商。以请求的首部字段为参考，在服务器端自动处理。
	- 客户端驱动协商（Agent-driven Negotiation）
		由客户端进行内容协商的方式。用
	- 透明协商（Transparent Negotiation）
		是服务器驱动和客户端驱动的结合体，是由服务器端和客户端各自进行内容协商的一种方法。

## Web服务器

### 虚拟主机

HTTP/1.1 规范允许一台 HTTP 服务器搭建多个 Web 站点，因此利用**虚拟主机**可令一个IP地址配对多个URI
互联网通过DNS服务将URI解析为IP进行连接，连接至服务器上通过首部中的`Host`指定的URI确认具体的访问域名

### 转发服务器

#### 代理

接收客户端发送的请求后转发给其他服务器。代理*不改变请求 URI*，会直接发送给前方持有资源的目标服务器。
每次通过代理服务器转发请求或响应时，会追加写入`Via`首部信息

*用途*：利用缓存技术（稍后讲解）减少网络带宽的流量，组织内部针对特定网站的访问控制，以获取访问日志为主要目的，等等

*类型*：**缓存/非缓存代理**、**透明/非透明代理**
代理转发响应时，缓存代理（Caching Proxy）会预先将资源的副本（缓存）保存在代理服务器上
转发请求或响应时，不对报文做任何加工的代理类型被称为透明代理

#### 网关

转发其他服务器通信数据的服务器

*用途*：可以由 HTTP 请求转化为其他协议通信

#### 隧道

在相隔甚远的客户端和服务器两者之间进行中转，并保持双方通信连接的应用程序

*用途*：用 SSL 等加密手段进行通信，确保客户端能与服务器进行安全的通信。

### 缓存

**缓存服务器**是代理服务器的一种，并归类在缓存代理类型中，缓存可以存在于缓存服务器内或客户端浏览器中。判定缓存过期后，会向源服务器确认资源的有效性，若判断浏览器缓存失效，浏览器会再次请求新资源。

## HTTPS

针对HTTP存在的部分缺点进行改进形成的扩展协议，*HTTP+ 加密（SSL） + 认证 + 完整性保护=**HTTPS***（HTTPSecure，超文本传输安全协议）
通过和 **SSL**（Secure Socket Layer，安全套接层）或 **TLS**（Transport Layer Security，安全层传输协议）的组合使用， 加密 HTTP 的通信内容。
通常，HTTP 直接和 TCP 通信。当使用 SSL 时，则演变成先和 SSL 通信，再由 SSL 和 TCP 通信了。

![|525](https://image.jiang849725768.asia/2022/202211261654567.png)

### HTTP的安全问题

1. 通信使用明文（不加密），内容可能会被窃听
2. 不验证通信方的身份，因此有可能遭遇伪装
3. 无法证明报文的完整性，所以有可能已遭篡改

#### 加密

按TCP/IP 协议族的工作机制，通信内容在所有的通信线路上都有可能遭到窥视。

加密方法：
- 通信加密 - 通过和 **SSL**（Secure Socket Layer，安全套接层）或 **TLS**（Transport Layer Security，安全层传输协议）的组合使用， 加密 HTTP 的通信内容。
- 内容加密 - 对报文主体进行加密（仍可能发生内容被篡改）

即使已经过加密处理的通信，也会被窥视到通信内容，经过加密的通信一定程度上可以让人无法破解内容中报文信息的含义

#### 验证身份

HTTP 协议中的请求和响应不会对通信方进行确认，导致问题：
- 无法确定请求发送至目标的 Web 服务器是否是按真实意图返回响应的那台服务器。有可能是已伪装的 Web 服务器。
- 无法确定响应返回到的客户端是否是按真实意图接收响应的那个客户端。有可能是已伪装的客户端。
- 无法确定正在通信的对方是否具备访问权限。因为某些Web 服务器上保存着重要的信息，只想发给特定用户通信的权限。
- 无法判定请求是来自何方、出自谁手。即使是无意义的请求也会照单全收。
- 无法阻止海量请求下的 DoS 攻击（Denial of Service，拒绝服务攻击）。

#### 验证报文完整性

请求或响应在传输途中，遭攻击者拦截并篡改内容的攻击称为**中间人攻击**（Man-in-the-Middle attack，MITM）

使用 HTTP 协议确定报文完整性常用的是 **MD5 和 SHA-1** 等算法生成**散列值**进行校验，以及用来确认文件的以 **PGP**（PrettyGood Privacy，完美隐私）创建的**数字签名**校验
HTTP下用户需要亲自使用上述方法进行验证，同时无法意识到PGP 和 MD5 本身的篡改

### SSL

HTTPS 使用 **SSL**（Secure Socket Layer） 和 **TLS**（Transport LayerSecurity）这两个协议。
SSL 技术最初是由浏览器开发商网景通信公司率先倡导的，开发过 SSL3.0 之前的版本。目前主导权已转移到 IETF（InternetEngineering Task Force，Internet 工程任务组）的手中。
IETF 以 SSL3.0 为基准，后又制定了 TLS1.0、TLS1.1 和TLS1.2。*TSL 是以 SSL 为原型开发的协议，有时会统一称该协议为 SSL。*当前主流的版本是 SSL3.0 和 TLS1.0。

**公开密钥加密**（Public-key cryptography）使用一对非对称的密钥。一把叫做**私有密钥**（private key），另一把叫做**公开密钥**（public key）。
发送密文的一方使用对方的公开密钥进行加密处理，对方收到被加密的信息后，再使用自己的私有密钥进行解密。

加密和解密同用一个密钥的方式称为**共享密钥加密**（Common keycrypto system），也被叫做**对称密钥加密**。

HTTPS 采用共享密钥加密和公开密钥加密两者并用的混合加密机制。
公开密钥加密与共享密钥加密相比，其处理速度要慢。因此在交换密钥环节使用公开密钥加密方式，之后的建立通信交换报文阶段则使用共享密钥加密方式。

#### 证书

采用由**数字证书认证机构**（CA，CertificateAuthority）和其相关机关颁发的**公开密钥证书**来验证公开密钥

数字证书认证机构为客户端与服务器双方都可信赖的第三方机构，对已申请的公开密钥做数字签名，将该公开密钥放入公钥证书后绑定在一起，然后分配该公开密钥
客户端使用数字证书认证机构的公开密钥，对证书上的数字签名进行验证
多数浏览器开发商事先在浏览器内部植入常用认证机关的公开密钥

*可确认对方服务器背后运营的企业是否真实存在*的证书是 **EV SSL 证书**（Extended Validation SSLCertificate）

安全性极高的认证机构可颁发**客户端证书**但仅用于特殊用途的业务，如网银系统。*客户端证书只能用来证明客户端实际存在，而不能用来证明用户本人的真实有效性*。

使每个人都可以用开源程序OpenSSL构建属于自己的认证机构，从而自己给自己颁发服务器证书，*由自认证机构颁发的证书称为自签名证书*

#### 问题

当使用 SSL 时，HTTP的处理速度会变慢
SSL 的慢分两种。一种是指通信慢。另一种是指由于大量消耗CPU 及内存等资源，导致处理速度变慢。
和使用 HTTP 相比，网络负载可能会变慢 2 到 100 倍。除去和TCP 连接、发送 HTTP 请求 • 响应以外，还必须进行 SSL 通信，因此整体上处理通信量不可避免会增加。
另一点是 SSL 必须进行加密处理。在服务器和客户端都需要进行加密和解密的运算处理。因此从结果上讲，比起 HTTP 会更多地消耗服务器和客户端的硬件资源，导致负载增强。

### 通信机制

流程图解：
![](https://image.jiang849725768.asia/2022/202211261754805.png)
- CBC 模式（Cipher Block Chaining）又名密码分组链接模式

在以上流程中，应用层发送数据时会附加一种叫做 **MAC**（MessageAuthentication Code）的报文摘要。MAC 能够查知报文是否遭到篡改，从而保护报文的完整性。

## 用户身份认证

常见认证信息：密码、动态令牌、数字证书、生物认证、IC卡等

HTTP/1.1使用的认证方式：BASIC 认证（基本认证）、DIGEST 认证（摘要认证）、SSL 客户端认证、FormBase 认证（基于表单认证）等

### Basic 认证

![](https://image.jiang849725768.asia/2022/202211271047364.png)

- 安全性低，明文解码后就是用户 ID和密码
- 一般浏览器无法实现认证的注销操作

### DIGEST认证

![](https://image.jiang849725768.asia/2022/202211271054320.png)

1. ①中首部字段 `WWW-Authenticate` 内必须包含 `realm` 和 `nonce` 这两个字段的信息。客户端就是依靠向服务器回送这两个值进行认证的。
2. ②中首部字段 `Authorization` 内必须包含 `username`、`realm`、`nonce`、`uri` 和 `response` 的字段信息。
	- `realm` 和 `nonce` 就是之前从服务器接收到的响应中的字段。
	- `username` 是 `realm` 限定范围内可进行认证的用户名
	- `uri`（digest-uri）即 `Request-URI` 的值，经代理转发后`Request-URI`的值可能被修改，因此事先会复制一份副本保存在`uri`内。
	- `response` 也可叫做 `Request-Digest`，存放经过 MD5 运算后的密码字符串，形成响应码。

### SSL客户端认证

借由 HTTPS 的客户端证书完成认证

步骤：
1. 接收到需要认证资源的请求，服务器会发送 `CertificateRequest` 报文，要求客户端提供客户端证书。
2. 用户选择将发送的客户端证书后，客户端会把客户端证书信息以 `Client Certificate` 报文方式发送给服务器。
3. 服务器验证客户端证书验证通过后方可领取证书内客户端的公开密钥，然后开始 HTTPS 加密通信。

多数情况下和[[Work/CODE/Web/HTTP#表单认证\|表单认证]]组合形成一种**双因素认证**（Two-factor authentication）来使用

### 表单认证

基于表单的认证方法不是在 HTTP 协议中定义的。客户端会向服务器上的 Web 应用程序发送登录信息（Credential），按登录信息的验证结果认证。

*用户身份认证多半基于表单认证*
基于表单认证的标准规范尚未有定论，一般会使用 **Cookie** 来管理 **Session**（会话）。

1. 客户端使用HTTPS进行用户输入数据（用户名&密码）的发送
2. 服务器将用户认证状态和**Session ID**绑定后向用户发放包含Session ID的cookies
3. 之后客户端发送请求时服务器验证接收到的Cookies中的Session ID识别用户和其认证状态

服务器端应如何保存用户提交的密码等登录信息等也没有标准化，通常的安全做法是先利用给密码加盐（salt）的方式增加额外信息，再使用散列（hash）函数计算出散列值后保存。
- salt 是由服务器随机生成的一个长度足够长的字符串，和密码字符串相连接（前后都可以）生成散列值。
- 当两个用户使用了同一个密码时，由于随机生成的 salt 值不同，对应的散列值也将是不同的。

## HTTP发展

### WebSocket

>[!cite]  参考
>[WebSocket 教程 - 阮一峰的网络日志 (ruanyifeng.com)](https://www.ruanyifeng.com/blog/2017/05/websocket.html)

WebSocket，即 Web 浏览器与 Web 服务器之间**全双工通信**（Full Duplex，双向同时通信）标准。
WebSocket是建立在 HTTP 基础上的协议，连接的发起方仍是客户端，而一旦确立 WebSocket 通信连接，不论服务器还是客户端，任意一方都可直接向对方发送报文。
为了实现 WebSocket 通信，在 HTTP 连接建立之后，需要完成一次“握手”（Handshaking）的步骤。

![|600](https://image.jiang849725768.asia/2022/202211281511551.png)

WebSocket的最大特点就是，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，是真正的双向平等对话，属于服务器推送技术的一种。

其他特点包括：
- 建立在 TCP 协议之上，服务器端的实现比较容易。
- 与 HTTP 协议有着良好的兼容性。默认端口也是80和443，并且握手阶段采用 HTTP 协议，因此握手时不容易屏蔽，能通过各种 HTTP 代理服务器。
- 数据格式比较轻量，性能开销小，通信高效。
- 可以发送文本，也可以发送二进制数据。
- 没有同源限制，客户端可以与任意服务器通信。
- 协议标识符是`ws`（如果加密，则为`wss`），服务器网址就是 URL。

### WebDav

**WebDAV**（Web-based Distributed Authoring and Versioning，基于万维网的分布式创作和版本控制）是一个可对 Web 服务器上的内容直接进行文件复制、编辑等操作的分布式文件系统。
WebDAV协议最重要的功能包括维护作者或修改日期的属性、名字空间管理、集合和覆盖保护。维护属性包括创建、删除和查询文件信息等。**名字空间管理**处理在服务器名称空间内复制和移动网页的能力。**集合**（Collections）处理各种资源的创建、删除和列举。**覆盖保护**处理与锁定文件相关的方面。

>[!note] 名字空间
>**名字空间**（英语：Namespace），也称**命名空间**、**名称空间**等，它表示着一个标识符（identifier）的可见范围。一个标识符可在多个名字空间中定义，它在不同名字空间中的含义是互不相干的。这样，在一个新的名字空间中可定义任何标识符，它们不会与任何已有的标识符发生冲突，因为已有的定义都处于其他名字空间中。

新增特有概念：
- **集合**（Collection）：是一种统一管理多个资源的概念。以集合为单位可进行各种操作。也可实现类似集合的集合这样的叠加。
- **资源**（Resource）：把文件或集合称为资源。
- **属性**（Property）：定义资源的属性。定义以“名称 = 值”的格式执行。
- **锁**（Lock）：把文件设置成无法编辑状态。多人同时编辑时，可防止在同一时间进行内容写入

新增方法：
- PROPFIND ：获取属性
- PROPPATCH ：修改属性
- MKCOL ：创建集合
- COPY ：复制资源及属性
- MOVE ：移动资源
- LOCK ：资源加锁
- UNLOCK ：资源解锁

新增状态码：
- 102 Processing ：可正常处理请求，但目前是处理中状态
- 207 Multi-Status ：存在多种状态
- 422 Unprocessible Entity ：格式正确，内容有误
- 423 Locked ：资源已被加锁
- 424 Failed Dependency ：处理与某请求关联的请求失败，因此不再维持依赖关系
- 507 Insufficient Storage ：保存空间不足

### HTTP/2

> [!cite]  参考
> [HTTP/2 - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/HTTP/2)
> [再过五分钟，你就懂 HTTP 2.0 了！_程序员cxuan的博客-CSDN博客](https://cxuan.blog.csdn.net/article/details/120037166)
> [HTTP/2 相比 1.0 有哪些重大改进？ - leozhang2018的回答 - 知乎](https://www.zhihu.com/question/34074946/answer/75364178)
>[Http系列(二) Http2中的多路复用 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903935648497678)

**HTTP/2**（超文本传输协议第2版，最初命名为**HTTP 2.0**），简称为**h2**（基于TLS/1.2或以上版本的加密连接）或**h2c**（非加密连接），是HTTP协议的的第二个主要版本，使用于万维网。
HTTP/2的设计本身允许非加密的HTTP协议，也允许使用TLS 1.2或更新版本协议进行加密。协议本身未要求必须使用加密， 但多数客户端（例如Firefox、Chrome、Safari、Opera、IE和Edge等）的开发者声明，他们只会实现通过TLS加密的HTTP/2协议，这使得经TLS加密的HTTP/2成为了事实上的强制标准，而*h2c事实上被主流浏览器废弃*。

#### 起源: SPDY

HTTP的瓶颈：**带宽**和**延迟**
*HTTP 1.1存在的问题*：
- 一条连接上只可发送一个请求。
- 请求只能从客户端开始。客户端不可以接收除响应以外的指令。
- 请求 / 响应首部未经压缩就发送。首部信息越多延迟越大。
- 发送冗长的首部。每次互相发送相同的首部造成的浪费较多。
- 管线化（pipelining）未充分解决阻塞问题
	- 浏览器客户端在同一时间，针对同一域名下的请求有一定数量限制。超过限制数目的请求会被阻塞
	- 只有幂等的请求比如 GET、HEAD 才能使用 pipelining ，非幂等请求比如 POST 则不能使用，因为请求之间可能存在先后依赖关系。
	- 队头阻塞问题并没有完全解决，因为服务器返回的响应还是要依请求顺序先发先回。
	- 绝大多数 HTTP 代理服务器不支持 pipelining。
	- 和不支持 pipelining 的老服务器协商有问题。

>[!note] 幂等
>假如在不考虑诸如错误或者过期等问题的情况下，若干次请求的副作用与单次请求相同或者根本没有副作用，那么这些请求方法就能够被视作“**幂等**(idempotence)”的。GET，HEAD，PUT和DELETE方法都有这样的幂等属性，同样由于根据协议，OPTIONS，TRACE都不应有副作用，因此也理所当然也是幂等的。

Google 在 2010 年发布了 **SPDY**（取自 SPeeDY，发音同 speedy），SPDY 的目标在于解决 HTTP 的缺陷，即延迟和安全性。
SPDY在 TCP/IP 的应用层与运输层之间通过新加会话层的形式运作。同时，考虑到安全性问题，SPDY 规定通信中使用 SSL。

*SPDY功能*：
- 多路复用流 - 通过单一的 TCP 连接，可以无限制处理多个 HTTP 请求。
- 服务器主动推送数据
- 压缩HTTP首部
- 赋予请求优先级
- 服务器提示 - 服务器可以主动提示客户端请求所需的资源。

#### HTTP/2的变化

*基于普适性的重要前提*：
- 客户端向服务器发送请求的这种基本模型不会改变。
- 原有的协议头不会改变，使用 http:// 和 https:// 的服务和应用不会做任何修改，不会有 http2://。
- 使用 HTTP 1.x 的客户端和服务器可以平滑升级到 HTTP 2.0 上。
- 不识别 HTTP 2.0 的代理服务器可以将请求降级到 HTTP 1.x
	- 因此HTTP/2创建了一个协商协议标准，即**应用层协议协商**（ALPN），以便客户端能够从HTTP/1.0、HTTP/1.1、HTTP/2乃至其他非HTTP协议中做出选择，ALPN是一个TLS的扩展。

*重要特点*：
- **二进制分帧**
	在应用层(HTTP/2)和传输层(TCP or UDP)之间增加一个二进制分帧层，在二进制分帧层中，HTTP/2会将所有传输的信息分割为更小的消息和帧(frame),并对它们采用二进制格式的编码，其中HTTP1,X的首部信息会被封装到HEADER frame，而相应的内容实体则封装到DATA frame里。
	![](https://image.jiang849725768.asia/2022/202211272107027.png)
- **多路复用**
	HTTP/2与SPDY一样，所有请求都是通过一个 TCP 连接并发完成，将一个TCP连接分为若干个stream，每个stream中可以传输若干message，每个message由若干二进制frame组成，接收方根据不同frame首部的 stream id 标识符重新连接将不同的stream进行组装。
	HTTP/2可以对不同的 stream 设置不同的优先级，stream 之间也可以设置依赖，依赖和优先级都可以动态调整，*解决关键请求被阻塞的问题*
- **首部压缩**
	HTTP/2使用 **HPACK 压缩算法**来减少传输的 header 大小，同时通信双方各自缓存一份 header 字段表，这样能够避免重复传输 header ，也能够减小传输的大小
- **服务端推送**
	服务端向客户端发送比客户端请求更多的数据。这允许服务器直接提供浏览器渲染页面所需资源，而无须浏览器在收到、解析页面后再提起一轮请求，节约了加载时间。
	当服务端需要主动推送某个资源时，便会发送一个 Frame Type 为 `PUSH_PROMISE` 的 frame ，里面带了 PUSH 需要新建的 stream id。

> [!note]  帧&流
> **帧**（Frame）：
> - HTTP/2 中**数据传输的最小单位**，
> - 每一帧都包含几个字段，有**length、type、flags、stream identifier、frame playload**等，其中type 代表帧的类型，在 HTTP/2 的标准中定义了 10 种不同的[类型](https://webconcepts.info/concepts/http2-frame-type/)
> - 传输时HEADERS 帧在 DATA 帧前面
>
> **流**（Stream）：
> - 存在于连接中的一个虚拟通道。流可以承载双向消息，每个流都有一个唯一的整数**Stream ID**
> - **双向性**：同一个流内，可同时发送和接受数据。
> - **有序性**：流中被传输的数据就是二进制帧 。帧在流上的被发送与被接收都是按照顺序进行的。
> - **并行性**：流中的二进制帧 都是被并行传输的，无需按顺序等待。
> - 流的创建：流可以被客户端或服务器单方面建立, 使用或共享。
> - 流的关闭：流也可以被任意一方关闭。
> - 为了防止两端 stream id 冲突，客户端发起的流具有奇数 id，服务器端发起的流具有偶数 id。

*缺点*：
- 应用层协议协商导致的一次额外来回通信延迟（RTT）
- 在 HTTP 2.0 中，多个请求是在同一个 TCP 管道中，出现丢包时整个 TCP 都要开始等待重传，此时该TCP连接中的所有请求被阻塞

### HTTP/3

> [!cite]  参考
> [HTTP/3 - 維基百科，自由的百科全書 (wikipedia.org)](https://zh.wikipedia.org/zh-hk/HTTP/3)
> [5分钟看懂HTTP3_文化 & 方法_Mehdi Zed_InfoQ精选文章](https://www.infoq.cn/article/whcobxfbgtphy7ijv1kp)

**HTTP/3**是第三个主要版本的HTTP协议。在HTTP/3中，将弃用TCP协议，改为使用基于UDP协议的QUIC协议实现。

