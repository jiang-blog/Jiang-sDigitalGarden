---
{"dg-publish":true,"permalink":"/Code/3.ComputerNetwork/CN.0a.应用层/","title":"应用层","noteIcon":""}
---


# 应用层

网络应用程序仅运行在端系统上，不涉及[[Code/3.ComputerNetwork/CN.0.计算机网络#^internetCore\|网络核心]]

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

网络应用依赖于进程端口的相互通信
在同一个主机内使用进程间通信机制通信，在不同主机上通过交换报文来通信
- 进程：在主机上运行的应用程序
- 客户端进程：发起通信的进程
- 服务器进程：等待通信的进程

> P2P 中也存在客户端进程和服务器进程

应用层通过IP地址和端口号确定传输双方的身份
每个主机具有唯一的IP地址
每个进程具有一个**端口号**(port number)，流行的应用协议分配有特定的端口号
TCP、UDP 分别实现了端口，因此两个协议的端口不会互相影响

**应用层协议**规定了不同端系统上的应用进程如何相互交换报文
- 交换的报文类型，例如请求报文和响应报文
- 各种报文类型的语法，如报文中的各个字段及这些字段是如何描述的
- 字段的语义，即这些字段中的信息的含义
- 确定一个进程何时以及如何发送报文，对报文进行响应的规则

## 应用层相关协议

应用层通信协议包括 [[Code/3.ComputerNetwork/CN.0a1.HTTP协议\|HTTP协议]] 、[[Code/3.ComputerNetwork/CN.0a2.DNS协议\|DNS协议]] 和 [[Code/3.ComputerNetwork/CN.0a3.其他应用层协议\|RPC、FTP、Websocket 等其他协议]]

### URI

**URI**(Uniform Resource Identifier，统一资源定位标识符) 用字符串标识某一互联网资源，而 URL 表示资源的地点(互联网上所处的位置)，后者是前者的子集
表示指定的 URI，要使用涵盖全部必要信息的绝对 URI、绝对 URL 以及相对 URL

`http://user:pass@www.example.jp:80/dir/index.htm?uid=1#ch1` 为**绝对 URI**，不区分大小写
- `http` 或 `https` 为协议方案名
- `user:pass` 为登录信息(可选项)
	指定用户名和密码作为从服务器端获取资源时必要的登录信息
- `www.example.jp` 为服务器地址
	类似这种 DNS 可解析的名称，或是 192.168.1.1 这类 IPv4 地址名，还可以是\[0:0:0:0:0:0:0:1\] 这样用方括号括起来的 IPv6 地址名
- `80` 为服务器端口号(可选项)
	省略则自动使用默认端口号
- `/dir/index.htm` 为带层次的文件路径
	指定服务器上的文件路径来定位特指的资源
- `uid=1` 为查询字符串(可选项)
- `ch1` 为片段标识符(可选项)

相对 URL 与带层次的文件路径类似

用来制定 HTTP 协议技术标准的文档被称为**RFC**(征求修正意见书)

## Web 构建

Web 页由 HTML 文件，JPEG 图像等多种对象组成，其含有一个基本的 HTML 文件，该文件中包含对其他对象的引用(链接)

### HTML

**HTML**(HyperText Markup Language，超文本标记语言)是为了发送 Web 上的**超文本**(Hypertext)而开发的标记语言。
超文本是一种文档系统，可将文档中任意位置的信息与其他信息(文本或图片等)建立关联，即超链接文本。
标记语言是指通过在文档的某部分穿插特别的字符串标签，用来修饰文档的语言。
我们把出现在 HTML 文档内的这种特殊字符串叫做 **HTML 标签**(Tag)

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

HTML5 是 HTML 最新的修订版本，广义论及 HTML5 时，实际指的是包括 HTML、CSS 和 JavaScript 在内的一套技术组合。

#### 动态 HTML

**动态 HTML**(Dynamic HTML)是通过调用客户端脚本语言**JavaScript**将静态的 HTML 内容变成动态的技术的总称
利用 **DOM**(Document ObjectModel，文档对象模型)可指定欲发生动态变化的 HTML 元素

### XML

**XML**(eXtensible Markup Language，可扩展标记语言)是一种可按应用目标进行扩展的通用标记语言
XML 和 HTML 都是从标准通用标记语言 **SGML**(Standard GeneralizedMarkup Language)简化而成，与 HTML 相比，它对数据的记录方式做了特殊处理

### JSON

**JSON**(JavaScript Object Notation)是一种以 JavaScript(ECMAScript)的对象表示法为基础的轻量级数据标记语言。能够处理的数据类型有 *布尔值/空值/对象/数组/数字/字符串*，这 7 种类型。

### CSS

**CSS**(Cascading Style Sheets，层叠样式表)可用于指定 HTML 和 XML 内的各种元素的样式。
示例：
```css
.logo {
padding: 20px;
text-align: center;
}
```

### WEB 应用

由程序创建的 HTML 内容称为*动态内容*，而事先准备好的 HTML 内容称为*静态内容*

**CGI**(Common Gateway Interface，通用网关接口)是为提供网络服务而执行控制台应用 (或称命令行界面)的程序。在 CGI 的作用下，程序会对请求内容做出相应的动作，比如创建动态内容。

**Servlet** 是一种能在服务器上创建动态内容的程序，解决了 CGI 创建程序带来的服务器负担
Servlet 是用 Java 语言实现的一个接口，属于面向企业级 Java 的一部分

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

CND在缓存服务器节点中存储内容的多个拷贝，用户请求内容时重定向到其中一个，从其中获取内容
大多数 CDN 利用 DNS 来截获并重定向请求，在 DNS 解析时，CDN 内部的 DNS 智能调度系统通过负载均衡、网路等情况来选择出与主机最合适的资源服务器的 IP 作为 DNS 解析结果

CDN 有两个关键组成部分：**全局负载均衡**和**缓存系统**

全局负载均衡（Global Sever Load Balance）一般简称为 GSLB，它是 CDN 的“大脑”，主要的职责是当用户接入网络的时候在 CDN 专网中挑选出一个“最佳”节点提供服务，解决的是用户如何找到“最近的”边缘节点，对整个 CDN 网络进行“负载均衡”

缓存系统有选择地缓存那些最常用的那些资源
