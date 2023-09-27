---
{"dg-publish":true,"permalink":"/Code/3.ComputerNetwork/CN.0a1b.HTTPS/","title":"CN.0a1b.HTTPS","noteIcon":""}
---


# HTTPS

针对 HTTP 存在的部分缺点进行改进形成的扩展协议，**HTTP+ 加密(SSL) + 认证 + 完整性保护=*HTTPS***(超文本传输安全协议)
HTTPS 通过 *SSL*(安全套接层)/*TLS*(安全层传输协议)加密 HTTP 的通信内容

> *TSL* 是以 SSL 为原型开发的协议，有时会统一称该协议为 SSL/TLS

<svg xmlns="http://www.w3.org/2000/svg" version="1.1" height="261px" width="351px" viewBox="-10 -10 371 281" content="&lt;mxGraphModel dx=&quot;1243&quot; dy=&quot;487&quot; grid=&quot;1&quot; gridSize=&quot;10&quot; guides=&quot;1&quot; tooltips=&quot;1&quot; connect=&quot;1&quot; arrows=&quot;1&quot; fold=&quot;1&quot; page=&quot;0&quot; pageScale=&quot;1&quot; pageWidth=&quot;827&quot; pageHeight=&quot;1169&quot; math=&quot;0&quot; shadow=&quot;0&quot;&gt;&lt;root&gt;&lt;mxCell id=&quot;0&quot;/&gt;&lt;mxCell id=&quot;1&quot; parent=&quot;0&quot;/&gt;&lt;mxCell id=&quot;2&quot; value=&quot;&quot; style=&quot;rounded=0;whiteSpace=wrap;html=1;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;1&quot;&gt;&lt;mxGeometry x=&quot;20&quot; y=&quot;180&quot; width=&quot;350&quot; height=&quot;260&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;3&quot; value=&quot;&quot; style=&quot;shape=table;html=1;whiteSpace=wrap;startSize=0;container=1;collapsible=0;childLayout=tableLayout;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;1&quot;&gt;&lt;mxGeometry x=&quot;50&quot; y=&quot;200&quot; width=&quot;110&quot; height=&quot;150&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;4&quot; value=&quot;&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;collapsible=0;dropTarget=0;pointerEvents=0;fillColor=none;top=0;left=0;bottom=0;right=0;points=[[0,0.5],[1,0.5]];portConstraint=eastwest;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;3&quot;&gt;&lt;mxGeometry width=&quot;110&quot; height=&quot;50&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;5&quot; value=&quot;HTTP&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;connectable=0;fillColor=#dae8fc;top=0;left=0;bottom=0;right=0;overflow=hidden;pointerEvents=1;fontSize=14;strokeColor=#6c8ebf;&quot; vertex=&quot;1&quot; parent=&quot;4&quot;&gt;&lt;mxGeometry width=&quot;110&quot; height=&quot;50&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;8&quot; value=&quot;&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;collapsible=0;dropTarget=0;pointerEvents=0;fillColor=none;top=0;left=0;bottom=0;right=0;points=[[0,0.5],[1,0.5]];portConstraint=eastwest;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;3&quot;&gt;&lt;mxGeometry y=&quot;50&quot; width=&quot;110&quot; height=&quot;50&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;9&quot; value=&quot;TCP&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;connectable=0;fillColor=none;top=0;left=0;bottom=0;right=0;overflow=hidden;pointerEvents=1;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;8&quot;&gt;&lt;mxGeometry width=&quot;110&quot; height=&quot;50&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;12&quot; value=&quot;&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;collapsible=0;dropTarget=0;pointerEvents=0;fillColor=none;top=0;left=0;bottom=0;right=0;points=[[0,0.5],[1,0.5]];portConstraint=eastwest;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;3&quot;&gt;&lt;mxGeometry y=&quot;100&quot; width=&quot;110&quot; height=&quot;50&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;13&quot; value=&quot;IP&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;connectable=0;fillColor=none;top=0;left=0;bottom=0;right=0;overflow=hidden;pointerEvents=1;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;12&quot;&gt;&lt;mxGeometry width=&quot;110&quot; height=&quot;50&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;16&quot; value=&quot;HTTP&quot; style=&quot;text;html=1;strokeColor=none;fillColor=none;align=center;verticalAlign=middle;whiteSpace=wrap;rounded=0;fontSize=17;fontStyle=1&quot; vertex=&quot;1&quot; parent=&quot;1&quot;&gt;&lt;mxGeometry x=&quot;77.5&quot; y=&quot;390&quot; width=&quot;55&quot; height=&quot;20&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;17&quot; value=&quot;&quot; style=&quot;shape=table;html=1;whiteSpace=wrap;startSize=0;container=1;collapsible=0;childLayout=tableLayout;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;1&quot;&gt;&lt;mxGeometry x=&quot;220&quot; y=&quot;200&quot; width=&quot;110&quot; height=&quot;150&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;18&quot; value=&quot;&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;collapsible=0;dropTarget=0;pointerEvents=0;fillColor=none;top=0;left=0;bottom=0;right=0;points=[[0,0.5],[1,0.5]];portConstraint=eastwest;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;17&quot;&gt;&lt;mxGeometry width=&quot;110&quot; height=&quot;38&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;19&quot; value=&quot;HTTP&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;connectable=0;fillColor=#dae8fc;top=0;left=0;bottom=0;right=0;overflow=hidden;pointerEvents=1;fontSize=14;strokeColor=#6c8ebf;&quot; vertex=&quot;1&quot; parent=&quot;18&quot;&gt;&lt;mxGeometry width=&quot;110&quot; height=&quot;38&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;24&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;collapsible=0;dropTarget=0;pointerEvents=0;fillColor=none;top=0;left=0;bottom=0;right=0;points=[[0,0.5],[1,0.5]];portConstraint=eastwest;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;17&quot;&gt;&lt;mxGeometry y=&quot;38&quot; width=&quot;110&quot; height=&quot;37&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;25&quot; value=&quot;SSL&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;connectable=0;fillColor=#60a917;top=0;left=0;bottom=0;right=0;overflow=hidden;pointerEvents=1;fontSize=14;strokeColor=#2D7600;fontColor=#ffffff;&quot; vertex=&quot;1&quot; parent=&quot;24&quot;&gt;&lt;mxGeometry width=&quot;110&quot; height=&quot;37&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;20&quot; value=&quot;&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;collapsible=0;dropTarget=0;pointerEvents=0;fillColor=none;top=0;left=0;bottom=0;right=0;points=[[0,0.5],[1,0.5]];portConstraint=eastwest;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;17&quot;&gt;&lt;mxGeometry y=&quot;75&quot; width=&quot;110&quot; height=&quot;38&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;21&quot; value=&quot;TCP&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;connectable=0;fillColor=none;top=0;left=0;bottom=0;right=0;overflow=hidden;pointerEvents=1;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;20&quot;&gt;&lt;mxGeometry width=&quot;110&quot; height=&quot;38&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;22&quot; value=&quot;&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;collapsible=0;dropTarget=0;pointerEvents=0;fillColor=none;top=0;left=0;bottom=0;right=0;points=[[0,0.5],[1,0.5]];portConstraint=eastwest;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;17&quot;&gt;&lt;mxGeometry y=&quot;113&quot; width=&quot;110&quot; height=&quot;37&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;23&quot; value=&quot;IP&quot; style=&quot;shape=partialRectangle;html=1;whiteSpace=wrap;connectable=0;fillColor=none;top=0;left=0;bottom=0;right=0;overflow=hidden;pointerEvents=1;fontSize=14;&quot; vertex=&quot;1&quot; parent=&quot;22&quot;&gt;&lt;mxGeometry width=&quot;110&quot; height=&quot;37&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;mxCell id=&quot;26&quot; value=&quot;HTTPS&quot; style=&quot;text;html=1;strokeColor=none;fillColor=none;align=center;verticalAlign=middle;whiteSpace=wrap;rounded=0;fontSize=17;fontStyle=1&quot; vertex=&quot;1&quot; parent=&quot;1&quot;&gt;&lt;mxGeometry x=&quot;247.5&quot; y=&quot;390&quot; width=&quot;55&quot; height=&quot;20&quot; as=&quot;geometry&quot;/&gt;&lt;/mxCell&gt;&lt;/root&gt;&lt;/mxGraphModel&gt;"><style type="text/css"></style><rect x="0.5" y="0.5" width="350" height="260" fill="#ffffff" stroke="#000000" pointer-events="none"/><rect x="30.5" y="20.5" width="110" height="150" fill="#ffffff" stroke="#000000" pointer-events="none"/><path d="M 30.5 70.5 L 140.5 70.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><path d="M 30.5 120.5 L 140.5 120.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><path d="M 30.5 20.5 M 140.5 20.5 M 140.5 70.5 M 30.5 70.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><rect x="30.5" y="20.5" width="110" height="50" fill="#dae8fc" stroke="none" pointer-events="none"/><path d="M 30.5 20.5 M 140.5 20.5 M 140.5 70.5 M 30.5 70.5" fill="none" stroke="#6c8ebf" stroke-miterlimit="10" pointer-events="none"/><g><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 46px; margin-left: 32px;"><div style="box-sizing: border-box; font-size: 0; text-align: center; max-height: 46px; overflow: hidden; "><div style="display: inline-block; font-size: 14px; font-family: Helvetica; color: #000000; line-height: 1.2; pointer-events: none; white-space: normal; word-wrap: normal; ">HTTP</div></div></div></foreignObject></g><path d="M 30.5 70.5 M 140.5 70.5 M 140.5 120.5 M 30.5 120.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><path d="M 30.5 70.5 M 140.5 70.5 M 140.5 120.5 M 30.5 120.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><g><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 96px; margin-left: 32px;"><div style="box-sizing: border-box; font-size: 0; text-align: center; max-height: 46px; overflow: hidden; "><div style="display: inline-block; font-size: 14px; font-family: Helvetica; color: #000000; line-height: 1.2; pointer-events: none; white-space: normal; word-wrap: normal; ">TCP</div></div></div></foreignObject></g><path d="M 30.5 120.5 M 140.5 120.5 M 140.5 170.5 M 30.5 170.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><path d="M 30.5 120.5 M 140.5 120.5 M 140.5 170.5 M 30.5 170.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><g><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 146px; margin-left: 32px;"><div style="box-sizing: border-box; font-size: 0; text-align: center; max-height: 46px; overflow: hidden; "><div style="display: inline-block; font-size: 14px; font-family: Helvetica; color: #000000; line-height: 1.2; pointer-events: none; white-space: normal; word-wrap: normal; ">IP</div></div></div></foreignObject></g><g><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 53px; height: 1px; padding-top: 221px; margin-left: 59px;"><div style="box-sizing: border-box; font-size: 0; text-align: center; "><div style="display: inline-block; font-size: 17px; font-family: Helvetica; color: #000000; line-height: 1.2; pointer-events: none; font-weight: bold; white-space: normal; word-wrap: normal; ">HTTP</div></div></div></foreignObject></g><rect x="200.5" y="20.5" width="110" height="150" fill="#ffffff" stroke="#000000" pointer-events="none"/><path d="M 200.5 58.5 L 310.5 58.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><path d="M 200.5 95.5 L 310.5 95.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><path d="M 200.5 133.5 L 310.5 133.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><path d="M 200.5 20.5 M 310.5 20.5 M 310.5 58.5 M 200.5 58.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><rect x="200.5" y="20.5" width="110" height="38" fill="#dae8fc" stroke="none" pointer-events="none"/><path d="M 200.5 20.5 M 310.5 20.5 M 310.5 58.5 M 200.5 58.5" fill="none" stroke="#6c8ebf" stroke-miterlimit="10" pointer-events="none"/><g><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 40px; margin-left: 202px;"><div style="box-sizing: border-box; font-size: 0; text-align: center; max-height: 34px; overflow: hidden; "><div style="display: inline-block; font-size: 14px; font-family: Helvetica; color: #000000; line-height: 1.2; pointer-events: none; white-space: normal; word-wrap: normal; ">HTTP</div></div></div></foreignObject></g><path d="M 200.5 58.5 M 310.5 58.5 M 310.5 95.5 M 200.5 95.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><rect x="200.5" y="58.5" width="110" height="37" fill="#60a917" stroke="none" pointer-events="none"/><path d="M 200.5 58.5 M 310.5 58.5 M 310.5 95.5 M 200.5 95.5" fill="none" stroke="#2d7600" stroke-miterlimit="10" pointer-events="none"/><g><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 77px; margin-left: 202px;"><div style="box-sizing: border-box; font-size: 0; text-align: center; max-height: 33px; overflow: hidden; "><div style="display: inline-block; font-size: 14px; font-family: Helvetica; color: #ffffff; line-height: 1.2; pointer-events: none; white-space: normal; word-wrap: normal; ">SSL</div></div></div></foreignObject></g><path d="M 200.5 95.5 M 310.5 95.5 M 310.5 133.5 M 200.5 133.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><path d="M 200.5 95.5 M 310.5 95.5 M 310.5 133.5 M 200.5 133.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><g><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 115px; margin-left: 202px;"><div style="box-sizing: border-box; font-size: 0; text-align: center; max-height: 34px; overflow: hidden; "><div style="display: inline-block; font-size: 14px; font-family: Helvetica; color: #000000; line-height: 1.2; pointer-events: none; white-space: normal; word-wrap: normal; ">TCP</div></div></div></foreignObject></g><path d="M 200.5 133.5 M 310.5 133.5 M 310.5 170.5 M 200.5 170.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><path d="M 200.5 133.5 M 310.5 133.5 M 310.5 170.5 M 200.5 170.5" fill="none" stroke="#000000" stroke-miterlimit="10" pointer-events="none"/><g><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 108px; height: 1px; padding-top: 152px; margin-left: 202px;"><div style="box-sizing: border-box; font-size: 0; text-align: center; max-height: 33px; overflow: hidden; "><div style="display: inline-block; font-size: 14px; font-family: Helvetica; color: #000000; line-height: 1.2; pointer-events: none; white-space: normal; word-wrap: normal; ">IP</div></div></div></foreignObject></g><g><foreignObject style="overflow: visible; text-align: left;" pointer-events="none" width="100%" height="100%"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 53px; height: 1px; padding-top: 221px; margin-left: 229px;"><div style="box-sizing: border-box; font-size: 0; text-align: center; "><div style="display: inline-block; font-size: 17px; font-family: Helvetica; color: #000000; line-height: 1.2; pointer-events: none; font-weight: bold; white-space: normal; word-wrap: normal; ">HTTPS</div></div></div></foreignObject></g></svg>

## HTTP 的安全问题

- **窃听风险** - 通信使用明文(不加密)，内容可能会被窃听
- **冒充风险** - 不验证通信方的身份，因此有可能遭遇伪装
- **篡改风险** - 无法证明报文的完整性，所以有可能已遭篡改

按TCP/IP 协议族的工作机制，通信内容在所有的通信线路上都有可能遭到窥视

HTTP 协议中的请求和响应不会对通信方进行确认，导致问题：
- 无法确定请求发送至目标的 Web 服务器是否是按真实意图返回响应的那台服务器。有可能是已伪装的 Web 服务器
- 无法确定响应返回到的客户端是否是按真实意图接收响应的那个客户端。有可能是已伪装的客户端
- 无法确定正在通信的对方是否具备访问权限。因为某些Web 服务器上保存着重要的信息，只想发给特定用户通信的权限
- 无法判定请求是来自何方、出自谁手。即使是无意义的请求也会照单全收
- 无法阻止海量请求下的 DoS 攻击(拒绝服务攻击)

请求或响应在传输途中，遭攻击者拦截并篡改内容的攻击称为**中间人攻击**(MITM)
使用 HTTP 协议确定报文完整性常用的是 **哈希算法生成散列值**进行校验，以及用来确认文件的以 *PGP*创建的*数字签名*校验
HTTP 下用户需要亲自使用上述方法进行验证，同时无法意识到针对 PGP 和 MD5的篡改

## HTTPS 安全保证

SSL/TLS 协议可以很好地解决上述风险：
- **信息加密**：交互信息无法被窃取
- **校验机制**：无法篡改通信内容，篡改了就不能正常显示
- **身份证书**：网站的身份可以得到证明

具体的操作方式：
- **混合加密**的方式实现信息的**机密性**，解决了窃听的风险
- **摘要算法**的方式来实现**完整性**，它能够为数据生成独一无二的「指纹」，指纹用于校验数据的完整性，解决了篡改的风险
- 将服务器公钥放入到**数字证书**中，解决了冒充的风险

### 混合加密

- *非对称加密*又称*公开密钥加密*，使用一对非对称的密钥(*私有密钥*&*公开密钥*)进行加密，两个密钥可以双向加解密
  - 公钥加密，私钥解密：**保证内容传输的安全**，因为被公钥加密的内容，其他人是无法解密的，只有持有私钥的人，才能解密出实际的内容
  - 私钥加密，公钥解密：**保证消息不会被冒充**，因为私钥是不可泄露的，如果公钥能正常解密出私钥加密的内容，就能证明这个消息是来源于持有私钥身份的人发送的
*- 对称加密*也被叫做共享密钥加密，加密和解密同用一个密钥

- 对称加密只使用一个密钥，运算速度快，密钥必须保密，无法做到安全的密钥交换。
- 非对称加密使用两个密钥：公钥和私钥，公钥可以任意分发而私钥保密，解决了密钥交换问题但速度慢。

HTTPS 采用**共享密钥加密和公开密钥加密两者并用**的*混合加密*机制
- 建立通信时交换密钥环节使用公开密钥非对称加密
- 通信过程中交换报文阶段则使用共享密钥加密

### 摘要算法&数字签名

应用层发送数据时会附加用摘要算法(哈希函数)生成的报文摘要
通过比对报文摘要能够查知报文是否遭到篡改，从而保护报文的完整性

但是摘要算法不保证报文内容和哈希值没有被中间人整体替换，可以通过**私钥加密报文摘要，公钥解密**的方法确认消息来源可信，这种方法又被称为*数字签名算法*

### 身份证书

公开密钥的有效性通过由数字证书认证机构和其相关机关颁发的*公开密钥证书*来验证

*数字证书认证机构*(CA)为可信赖的第三方机构
1. 首先 CA 会把持有者的公钥、用途、颁发者、有效时间等信息打包，然后对这些信息进行 Hash 计算，得到一个 Hash 值
2. 然后 CA 会使用自己的私钥将该 Hash 值加密，生成 Certificate Signature，也就是数字签名
3. 最后将 Certificate Signature 添加在文件上，形成数字证书，然后分配该公开密钥
4. CA 使用自己的私钥对服务端申请的公开密钥做数字签名，将该公开密钥放入公钥证书后绑定在一起

**多数浏览器内部事先植入常用CA的公开密钥**
1. 客户端使用 CA 的公开密钥验证服务端数字证书上的签名
2. 验证成功后客户端从数字证书获取服务端公钥，加密报文后发送
3. 服务端使用私钥解密报文获取报文

> ![](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C/HTTP/22-%E6%95%B0%E5%AD%97%E8%AF%81%E4%B9%A6%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B.png)

证书的验证过程中还存在一个证书信任链的问题，因为我们向 CA 申请的证书一般不是根证书签发的，而是由中间证书签发的
证书链的目的是为了确保根证书的绝对安全性，将根证书隔离地越严格越好，不然根证书如果失守了，那么整个信任链都会有问题
> ![证书链.png (1478×452) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/https/%E8%AF%81%E4%B9%A6%E9%93%BE.png)

安全性极高的认证机构可颁发*客户端证书*但仅用于特殊用途的业务，如网银系统
客户端证书**只能用来证明客户端实际存在，而不能用来证明用户本人的真实有效性**

*EV SSL 证书*可确认对方服务器背后运营的企业是否真实存在
每个人都可以用开源程序OpenSSL构建属于自己的认证机构，从而自己给自己颁发服务器证书，由自认证机构颁发的证书称为**自签名证书**

## TLS 握手流程

基本流程：
1. 客户端向服务器索要并验证服务器的公钥
2. 双方协商生产会话秘钥
3. 双方采用会话秘钥进行加密通信

![](https://image.jiang849725768.asia/2022/202211261754805.png)
- CBC 模式(Cipher Block Chaining)又名密码分组链接模式

### TLS1.2

TLS1.2 的握手阶段涉及四次通信，使用不同的密钥交换算法的流程不同，现在常用的密钥交换算法有两种：*RSA 算法* 和 *ECDHE 算法*

#### RSA

RSA 算法握手具体流程：
1. 第一次握手，客户端向服务器发起加密通信  `Client Hello`  请求
        - 客户端支持的 TLS 协议版本
        - 客户端生产的随机数 `Client Random`
        - 客户端支持的密码套件列表，如 RSA 加密算法
2. 第二次握手，服务端响应请求
    1. `Sever Hello`
          - 确认支持 TLS 协议的版本(不支持时关闭加密通信)
          - 服务器生产的随机数 `Server Random`
          - 确认的密码套件，如 RSA 加密算法
    2. `Server Hello Done`
        - 告知客户端内容发送完毕
    3. `Server Certificate`
        - 服务器的数字证书
3. 客户端通过 CA 公钥验证数字证书，取出服务器公钥，生成一个随机数 `pre-master`，进而通过协商的加密算法生成会话密钥
4. 第三次握手，客户端向服务器发送报文
    1. `Cilent Key Exchange`
        - 以服务器公钥加密的随机数 `pre-master`
    2. `Change Cipher Spec`
        - 加密通信算法改变通知，表示随后的信息都将用会话秘钥加密通信
    3. `Encrypted Handshake Message（Finishd)`
        - 客户端握手结束通知，表示客户端的握手阶段已经结束，同时将之前所有内容生成摘要，用会话密钥加密供服务端校验
5. 服务器通过协商的加密算法计算出本次通信的会话秘钥
6. 第四次握手，服务器响应报文
    1. `Change Cipher Spec`
        - 加密通信算法改变通知，表示随后的信息都将用会话秘钥加密通信
    2. `Encrypted Handshake Message`
        - 服务器握手结束通知，表示服务器的握手阶段已经结束，同时将之前所有内容生成加密摘要供客户端校验

服务器和客户端**用双方协商的加密算法对过程中交换的三个随机数 `Client Random`、`Server Random`、`pre-master key` 进行加密**，各自生成相同的*会话秘钥(共享密钥)* 用于本次通信的对称加密

基于 RSA 算法的 HTTPS 存在**前向安全问题**：**如果服务端的私钥泄漏了，过去被第三方截获的所有 TLS 通讯密文都会被破解**

#### ECDHE

> [!Note] DH
> ![离散对数.png (692×227) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/https/%E7%A6%BB%E6%95%A3%E5%AF%B9%E6%95%B0.png)
> 通过离散对数，确认 a 与 p 值后，以 i 为私钥可以计算出公钥 b，目前的计算机能力无法做到由 b 反推 i
> 客户端与服务端交流确认相同的 a 与 p 值，各自生成私钥 $i_c$、$i_s$，进而计算出各自公钥 $b_c$、$b_s$，交换公钥后可以计算出相同的结果 K
> $$ K={b_s}^{i_c}(mod\ p)=(a^{i_s})^{i_c}(mod\ p)=(a^{i_c})^{i_s}(mod\ p)= {b_c}^{i_s}(mod\ p)$$
> K 即为客户端与服务端之间的**对称加密密钥**，可以作为会话密钥使用

**为实现前向安全，每次通信随机生成私钥**，但计算量大，性能不佳
*ECDHE 算法*通过 ECC 椭圆曲线特性，可以用更少的计算量计算出公钥，以及最终的会话密钥
1. 双方事先确定好使用哪种椭圆曲线，和曲线上的基点 G，这两个参数公开不加密
2. 双方各自随机生成一个随机数作为私钥 d，并与基点 G 相乘得到公钥 `Q = dG`，此时客户端的公私钥为 Q1 和 d1，服务器的公私钥为 Q2 和 d2
3. 双方交换各自的公钥，最后客户端计算点 `(x1，y1) = d1Q2`，服务器计算点 `(x2，y2) = d2Q1`，由于椭圆曲线满足乘法交换和结合律，所以 `d1Q2 = d1d2G = d2d1G = d2Q1` ，因此双方的 x 坐标是一样的，所以它是共享密钥，也就是会话密钥

ECDHE 算法握手具体流程：
1. 第一次握手
    1. 客户端向服务器发起加密通信 ` Client Hello ` 请求
        - 客户端支持的 TLS 协议版本
        - 客户端生产的随机数 `Client Random`
        - 客户端支持的密码套件列表
2. 第二次握手，**服务器生成随机数作为服务端椭圆曲线的私钥
    1. `Sever Hello`
        - 确认支持 TLS 协议的版本(不支持时关闭加密通信)
        - 服务器生产的随机数 `Server Random`
        - 确认的密码套件
    2. `Server Certificate`
        - 服务器的数字证书
    3. `Server Key Exchange`
        - 选择的椭圆曲线类型(同时确定了基点G)
        - 根据基点 G 和私钥计算出的服务端的椭圆曲线公钥(用 RSA 签名算法进行数字签名)
    4. `Server Hello Done`
        - 告知客户端内容发送完毕
3. 客户端通过 CA 公钥验证服务器身份，**生成一个随机数作为客户端椭圆曲线的私钥，进而生成客户端的椭圆曲线公钥**，同时通过协商的加密算法生成会话密钥
4. 第三次握手
    1. 客户端向服务器发送 `Cilent Key Exchange` 报文
        - 客户端椭圆曲线公钥
    2. 客户端向服务器发送 `Change Cipher Spec` 报文
        - 加密通信算法改变通知，表示随后的信息都将用会话秘钥加密通信
    3. 客户端向服务器发送 `Encrypted Handshake Message（Finishd)` 报文
        - 客户端握手结束通知，表示客户端的握手阶段已经结束，同时将之前所有内容生成摘要，用会话密钥加密供服务端校验
5. 服务器通过协商的加密算法计算出本次通信的会话秘钥
6. 第四次握手
    1. 服务器向客户端发送 `Change Cipher Spec` 报文
        - 加密通信算法改变通知，表示随后的信息都将用会话秘钥加密通信
    2. 服务器向客户端发送 `Encrypted Handshake Message` 报文
        - 服务器握手结束通知，表示服务器的握手阶段已经结束，同时将之前所有内容生成加密摘要供客户端校验

### TLS1.3

TLS 1.3 把 Hello 和公钥交换这两个消息合并成了一个消息，只需 1 RTT 就能完成 TLS 握手

> ![tls1.2and1.3.png (1832×1290) (xiaolincoding.com)](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/https%E4%BC%98%E5%8C%96/tls1.2and1.3.png)

具体做法：
1. 客户端在第一次握手时的 `Client Hello` 消息里带上了支持的椭圆曲线以及这些椭圆曲线对应的公钥
2. 服务端收到后，选定一个椭圆曲线等参数，然后返回消息时，带上服务端生成的公钥
3. 二者生成会话密钥进行加密通信

此外 TLS1.3 对于密钥交换算法废除了不支持前向安全性的 RSA 和 DH 算法，**只支持 ECDHE 算法**

### TLS 优化

#### 会话复用

*Session ID*
客户端和服务器首次 TLS 握手连接后，双方会在内存缓存会话密钥，并用唯一的 Session ID 来标识
当客户端再次连接时，hello 消息里会带上 Session ID，服务器收到后就会从内存找，如果找到就直接用该会话密钥恢复会话状态，跳过其余的过程，只用一个消息往返就可以建立安全通信
为了安全性，内存中的会话密钥会定期失效
缺点：
- 服务器必须保持每一个客户端的会话密钥，随着客户端的增多，**服务器的内存压力也会越大**
- 现在网站服务一般是由多台服务器通过负载均衡提供服务的，**客户端再次连接不一定会命中上次访问过的服务器**，于是还要走完整的 TLS 握手过程

*Session Ticket*
服务器不再缓存每个客户端的会话密钥，而是把缓存的工作交给了客户端
客户端与服务器首次建立连接时，服务器会加密「会话密钥」作为 Ticket 发给客户端，交给客户端缓存该 Ticket
客户端再次连接服务器时，客户端会发送 Ticket，服务器解密后就可以获取上一次的会话密钥，然后验证有效期，如果没问题，就可以恢复会话了，开始加密通信

对于集群服务器的话，要确保每台服务器加密会话密钥的密钥是一致的，这样客户端携带 Ticket 访问任意一台服务器时，都能恢复会话

Session ID 和 Session Ticket 都**不具备前向安全性**，一旦加密会话密钥的密钥被破解或者服务器泄漏会话密钥，前面劫持的通信密文都会被破解

同时应对*重放攻击*也很困难

*Pre-shared Key*

重连时，客户端会把 Ticket 和 HTTP 请求一同发送给服务端

Pre-shared Key 也有重放攻击的危险

> [!Note] 重放攻击
> ![重放攻击](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/%E7%BD%91%E7%BB%9C/https%E4%BC%98%E5%8C%96/%E9%87%8D%E6%94%BE%E6%94%BB%E5%87%BB.png)
> 假设 Alice 想向 Bob 证明自己的身份。 Bob 要求 Alice 的密码作为身份证明，爱丽丝应尽全力提供(可能是在经过如哈希函数的转换之后)。与此同时，Eve 窃听了对话并保留了密码(或哈希)
> 交换结束后，Eve(冒充 Alice )连接到 Bob。当被要求提供身份证明时，Eve 发送从 Bob 接受的最后一个会话中读取的 Alice 的密码(或哈希)，从而授予 Eve 访问权限
> 重放攻击的危险之处在于，如果中间人截获了某个客户端的 Session ID 或 Session Ticket 以及 POST 报文，而一般 POST 请求会改变数据库的数据，中间人就可以利用此截获的报文，不断向服务器发送该报文，这样就会导致数据库的数据被中间人改变了，而客户是不知情的
>避免重放攻击的方式就是需要**对会话密钥设定一个合理的过期时间**

## 应用数据完整性

TLS 在实现上分为*握手协议*和*记录协议*两层：
- TLS 握手协议就是我们前面说的 TLS 四次握手的过程，负责协商加密算法和生成对称密钥，后续用此密钥来保护应用程序数据
- TLS 记录协议负责**保护应用程序数据并验证其完整性和来源**，所以对 HTTP 数据加密是使用记录协议

具体流程：
1. 首先，消息被分割成多个较短的片段,然后分别对每个片段进行压缩
2. 接下来，经过压缩的片段会被加上消息认证码(哈希值)以保证完整性并进行数据的认证，通过附加消息认证码的 MAC 值，可以识别出篡改，与此同时，为了防止重放攻击，在计算消息认证码时，还加上了片段的编码
3. 再接下来，经过压缩的片段再加上消息认证码会一起通过对称密码进行加密
4. 最后，上述经过加密的数据再加上由数据类型、版本号、压缩后的长度组成的报头就是最终的报文数据

## HTTPS存在的问题

- 由于需要加密以及解密，HTTPS 的速度往往慢于 HTTP
- 进行SSL通信导致通信量的增加
- 服务端和客户端进行的加密解密消耗了时间，加大了主机的负担
