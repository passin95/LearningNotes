
<!-- TOC -->

- [一、基础概念](#%E4%B8%80%E5%9F%BA%E7%A1%80%E6%A6%82%E5%BF%B5)
  - [1.1 URI 和 URL](#11-uri-%E5%92%8C-url)
  - [1.2 TCP/IP](#12-tcpip)
  - [1.3 计算机网络结构](#13-%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%BD%91%E7%BB%9C%E7%BB%93%E6%9E%84)
    - [1.3.1 应用层](#131-%E5%BA%94%E7%94%A8%E5%B1%82)
    - [1.3.2 运输层](#132-%E8%BF%90%E8%BE%93%E5%B1%82)
      - [1.3.2.1 TCP](#1321-tcp)
      - [1.3.2.2 UDP](#1322-udp)
    - [1.3.3 网络层](#133-%E7%BD%91%E7%BB%9C%E5%B1%82)
    - [1.3.4 数据链路层](#134-%E6%95%B0%E6%8D%AE%E9%93%BE%E8%B7%AF%E5%B1%82)
    - [1.3.5 物理层](#135-%E7%89%A9%E7%90%86%E5%B1%82)
  - [1.4 Web 网页请求过程](#14-web-%E7%BD%91%E9%A1%B5%E8%AF%B7%E6%B1%82%E8%BF%87%E7%A8%8B)
- [二、HTTP](#%E4%BA%8Chttp)
  - [2.1 请求报文和响应报文](#21-%E8%AF%B7%E6%B1%82%E6%8A%A5%E6%96%87%E5%92%8C%E5%93%8D%E5%BA%94%E6%8A%A5%E6%96%87)
  - [2.2 请求方法](#22-%E8%AF%B7%E6%B1%82%E6%96%B9%E6%B3%95)
  - [2.3 HTTP 状态码](#23-http-%E7%8A%B6%E6%80%81%E7%A0%81)
    - [2.3.1 1XX 信息](#231-1xx-%E4%BF%A1%E6%81%AF)
    - [2.3.2 2XX 成功](#232-2xx-%E6%88%90%E5%8A%9F)
    - [2.3.3 3XX 重定向](#233-3xx-%E9%87%8D%E5%AE%9A%E5%90%91)
    - [2.3.4 4XX 客户端错误](#234-4xx-%E5%AE%A2%E6%88%B7%E7%AB%AF%E9%94%99%E8%AF%AF)
    - [2.3.5 5XX 服务器错误](#235-5xx-%E6%9C%8D%E5%8A%A1%E5%99%A8%E9%94%99%E8%AF%AF)
  - [2.4 HTTP 首部（请求头）](#24-http-%E9%A6%96%E9%83%A8%E8%AF%B7%E6%B1%82%E5%A4%B4)
    - [2.4.1 通用首部字段](#241-%E9%80%9A%E7%94%A8%E9%A6%96%E9%83%A8%E5%AD%97%E6%AE%B5)
    - [2.4.2 请求首部字段](#242-%E8%AF%B7%E6%B1%82%E9%A6%96%E9%83%A8%E5%AD%97%E6%AE%B5)
    - [2.4.3 响应首部字段](#243-%E5%93%8D%E5%BA%94%E9%A6%96%E9%83%A8%E5%AD%97%E6%AE%B5)
    - [2.4.4 实体首部字段](#244-%E5%AE%9E%E4%BD%93%E9%A6%96%E9%83%A8%E5%AD%97%E6%AE%B5)
  - [2.5 Content-Type](#25-content-type)
- [三、实际应用](#%E4%B8%89%E5%AE%9E%E9%99%85%E5%BA%94%E7%94%A8)
  - [3.1 Cookie](#31-cookie)
    - [3.1.1 用途](#311-%E7%94%A8%E9%80%94)
    - [3.1.2 创建过程](#312-%E5%88%9B%E5%BB%BA%E8%BF%87%E7%A8%8B)
    - [3.1.3 分类](#313-%E5%88%86%E7%B1%BB)
    - [3.1.4 JavaScript 获取 Cookie](#314-javascript-%E8%8E%B7%E5%8F%96-cookie)
    - [3.1.5 Secure 和 HttpOnly](#315-secure-%E5%92%8C-httponly)
    - [3.1.6 Session](#316-session)
    - [3.1.7 浏览器禁用 Cookie](#317-%E6%B5%8F%E8%A7%88%E5%99%A8%E7%A6%81%E7%94%A8-cookie)
    - [3.1.8 Cookie 与 Session 选择](#318-cookie-%E4%B8%8E-session-%E9%80%89%E6%8B%A9)
  - [3.2 缓存](#32-%E7%BC%93%E5%AD%98)
    - [3.2.1 作用](#321-%E4%BD%9C%E7%94%A8)
    - [3.2.2 实现方法](#322-%E5%AE%9E%E7%8E%B0%E6%96%B9%E6%B3%95)
    - [3.2.3 Cache-Control](#323-cache-control)
    - [3.2.4 缓存验证](#324-%E7%BC%93%E5%AD%98%E9%AA%8C%E8%AF%81)
  - [3.3 Authorization](#33-authorization)
    - [3.3.1 Basic](#331-basic)
    - [3.3.2 Bearer](#332-bearer)
      - [3.3.2.1 OAuth2 授权流程](#3321-oauth2-%E6%8E%88%E6%9D%83%E6%B5%81%E7%A8%8B)
      - [3.3.2.2 OAuth2 微信授权登陆流程](#3322-oauth2-%E5%BE%AE%E4%BF%A1%E6%8E%88%E6%9D%83%E7%99%BB%E9%99%86%E6%B5%81%E7%A8%8B)
      - [3.3.2.3 在自家 App 中使用 Bearer token](#3323-%E5%9C%A8%E8%87%AA%E5%AE%B6-app-%E4%B8%AD%E4%BD%BF%E7%94%A8-bearer-token)
      - [3.3.2.4 为什么引入 Authorization code](#3324-%E4%B8%BA%E4%BB%80%E4%B9%88%E5%BC%95%E5%85%A5-authorization-code)
      - [3.3.2.5 Refresh token](#3325-refresh-token)
  - [3.4 连接管理](#34-%E8%BF%9E%E6%8E%A5%E7%AE%A1%E7%90%86)
    - [3.4.1 短连接与长连接](#341-%E7%9F%AD%E8%BF%9E%E6%8E%A5%E4%B8%8E%E9%95%BF%E8%BF%9E%E6%8E%A5)
    - [3.4.2 流水线](#342-%E6%B5%81%E6%B0%B4%E7%BA%BF)
  - [3.5 内容协商](#35-%E5%86%85%E5%AE%B9%E5%8D%8F%E5%95%86)
    - [3.5.1 类型](#351-%E7%B1%BB%E5%9E%8B)
    - [3.5.2 Vary](#352-vary)
  - [3.6 内容编码](#36-%E5%86%85%E5%AE%B9%E7%BC%96%E7%A0%81)
  - [3.7 范围请求](#37-%E8%8C%83%E5%9B%B4%E8%AF%B7%E6%B1%82)
    - [3.7.1 Range](#371-range)
    - [3.7.2 Accept-Ranges](#372-accept-ranges)
    - [3.7.3 响应状态码](#373-%E5%93%8D%E5%BA%94%E7%8A%B6%E6%80%81%E7%A0%81)
  - [3.8 分块传输编码](#38-%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81)
  - [3.9 多部分对象集合](#39-%E5%A4%9A%E9%83%A8%E5%88%86%E5%AF%B9%E8%B1%A1%E9%9B%86%E5%90%88)
- [四、HTTPS](#%E5%9B%9Bhttps)
  - [4.1 加密](#41-%E5%8A%A0%E5%AF%86)
    - [4.1.1 对称密钥加密](#411-%E5%AF%B9%E7%A7%B0%E5%AF%86%E9%92%A5%E5%8A%A0%E5%AF%86)
    - [4.1.2 非对称密钥加密](#412-%E9%9D%9E%E5%AF%B9%E7%A7%B0%E5%AF%86%E9%92%A5%E5%8A%A0%E5%AF%86)
  - [4.2 证书（认证）](#42-%E8%AF%81%E4%B9%A6%E8%AE%A4%E8%AF%81)
    - [4.2.1 证书包含的信息](#421-%E8%AF%81%E4%B9%A6%E5%8C%85%E5%90%AB%E7%9A%84%E4%BF%A1%E6%81%AF)
    - [4.2.2 证书的验证过程](#422-%E8%AF%81%E4%B9%A6%E7%9A%84%E9%AA%8C%E8%AF%81%E8%BF%87%E7%A8%8B)
  - [4.3 完整性保护](#43-%E5%AE%8C%E6%95%B4%E6%80%A7%E4%BF%9D%E6%8A%A4)
  - [4.4 HTTPS 的缺点](#44-https-%E7%9A%84%E7%BC%BA%E7%82%B9)
  - [4.5 HTTPS 连接建立过程](#45-https-%E8%BF%9E%E6%8E%A5%E5%BB%BA%E7%AB%8B%E8%BF%87%E7%A8%8B)
- [五、GET 和 POST 的区别](#%E4%BA%94get-%E5%92%8C-post-%E7%9A%84%E5%8C%BA%E5%88%AB)
  - [5.1 作用](#51-%E4%BD%9C%E7%94%A8)
  - [5.2 参数](#52-%E5%8F%82%E6%95%B0)
  - [5.3 安全](#53-%E5%AE%89%E5%85%A8)
  - [5.4 幂等性](#54-%E5%B9%82%E7%AD%89%E6%80%A7)
  - [5.5 可缓存](#55-%E5%8F%AF%E7%BC%93%E5%AD%98)
- [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)

<!-- /TOC -->

# 一、基础概念

## 1.1 URI 和 URL

- URI（Uniform Resource Identifier，统一资源标识符）
- URL（Uniform Resource Locator，统一资源定位符）
- URN（Uniform Resource Name，统一资源名称），例如 urn:isbn:0-486-27557-4。

URI 包含 URL 和 URN，目前 WEB 只有 URL 比较流行，所以见到的基本都是 URL。

<div align="center"> <img src="../pictures//URI%20和%20URL%20的关系.webp" /> </div>

URI 的结构如下，它们都是等价的，只不过划分的区域不同：
```
[scheme:]scheme-specific-part[#fragment]
[scheme:][//authority][path][?query][#fragment]
[scheme:][//host:port][path][?query][#fragment]
```

以下用一个 URI 字符串来横向对比各部分结构的区别。

```
http://www.demo.com:8080/path/path1?key=value&key1=value1&key2=value2#demofragment
```

- scheme：URI的模式，例如 http、file、content 等，Deme 中为 http；
- scheme-specific-part：//www.demo.com:8080/path/path1?key=value&key1=value1&key2=value2；
- authority：www.demo.com:8080；
- host：URI的主机域名或IP地址，Demo 中为 www.demo.com；
- Part：端口号，Demo 中为 8080；
- path：路径信息，Demo 中为 /yourpath/fileName.htm；
- query：键对值，Deme 中为 key=value&key1=value1&key2=value2;
- fragment：用来标识次级资源，Demo 中为 demofragment。

## 1.2 TCP/IP

<div align="center"> <img src="../pictures//TCPIP.png" width="400"/> </div>

TCP/IP 是互联网相关的各类协议族的总称。

## 1.3 计算机网络结构

<div align="center"> <img src="../pictures//计算机网络体系结构.webp"/> </div>

### 1.3.1 应用层

应用层(application-layer）的任务是通过应用进程间的交互来完成特定网络应用。应用层协议定义的是应用进程（进程：主机中正在运行的程序）间的通信和交互的规则。对于不同的网络应用需要不同的应用层协议，例如域名系统 DNS，支持万维网应用的 HTTP 协议，支持电子邮件的 SMTP 协议等等。数据单元称为报文。

### 1.3.2 运输层

运输层主要提供数据的传输服务规则，例如如何保证数据的可靠传输以及拼接。由于应用层协议很多，定义通用的运输层协议就可以支持不断增多的应用层协议。运输层主要使用两种协议：传输控制协议 TCP 和用户数据报协议 UDP。

#### 1.3.2.1 TCP

提供面向连接、可靠的数据传输服务，数据单位为报文段。对收到的数据（例如 HTTP 请求报文）进行分割，并在各个报文上打上标记序号及端口号后转发给网络层。每一条 TCP 连接只能是点到点的。

#### 1.3.2.2 UDP

提供无连接、尽最大努力的数据传输服务，数据单位为用户数据报。UDP 具有较好的实时性，工作效率比 TCP 高，适用于对高速传输和实时性有较高的通信或广播通信。UDP 支持一对一，一对多，多对一和多对多的交互通信（多用于实时游戏）。

### 1.3.3 网络层

网络层只负责网络地址将源结点发出的数据包传送到目的结点。该层控制数据链路层与传输层之间的信息转发、建立、维持和终止网络的连接。该层一般是 IP 协议。

### 1.3.4 数据链路层 

数据链路层定义了在单个链路上如何传输数据，这些协议与被讨论的各种介质（WIFI、以太网）有关。数据链路层必须具备一系列相应的功能，主要有：如何将数据组合成数据块，在数据链路层中称这种数据块为帧，帧是数据链路层的传送单位；如何控制帧在物理信道上的传输，包括如何处理传输差错，如何调节发送速率以使与接收方相匹配；以及在两个网络实体之间提供数据链路通路的建立、维持和释放的管理。

### 1.3.5 物理层

在物理层上所传送的数据单位是比特。物理层的作用是实现相邻计算机节点之间比特流的透明传送，尽可能屏蔽掉具体传输介质和物理设备的差异。

## 1.4 Web 网页请求过程

<div align="center"> <img src="../pictures//TCP%20连接过程.webp"/> </div>

# 二、HTTP

- 超文本传输协议，现用于作为网络请求以及传输 HTML 内容、二进制数据等。
- HTTP 位于 TCP/IP 协议族中的最顶层———应用层

## 2.1 请求报文和响应报文

**（1）请求报文**

<div align="center"> <img src="../pictures//HTTP%20请求报文.png"/> </div><br>

**（2）响应报文**

<div align="center"> <img src="../pictures//HTTP%20响应报文.png"/> </div><br>

## 2.2 请求方法

**（1）GET**

- 用于获取资源。
- 不会修改服务器数据。
- 不发送 Body。

**（2）HEAD**

- 用于获取报文首部，即返回的响应没有 Body。

**（3）POST**

- 用于增加或修改资源。
- 传输内容写在 Body 中。

**（4）PUT**

- 向指定资源位置上传其最新内容。
- 传输内容写在 Body 中。

**（4）DELETE**

- 删除指定资源位置的资源。
- 请求报文没有请求体。

## 2.3 HTTP 状态码

服务器返回的 **响应报文** 中第一行为状态行，包含了状态码以及原因短语，用来告知客户端请求的结果。

| 状态码 | 类别 | 原因短语 |
| :---: | :---: | :---: |
| 1XX | Informational（信息性状态码） | 接收的请求正在处理 |
| 2XX | Success（成功状态码） | 请求正常处理完毕 |
| 3XX | Redirection（重定向状态码） | 需要进行附加操作以完成请求 |
| 4XX | Client Error（客户端错误状态码） | 服务器无法处理请求 |
| 5XX | Server Error（服务器错误状态码） | 服务器处理请求出错 |

### 2.3.1 1XX 信息

-  **100 Continue** ：表明到目前为止都很正常，客户端可以继续发送请求 (例如传输文件过大，需要分段传输，服务器返回 100 表示已经接受到客户端的需要，让客户端继续传输) 或者忽略这个响应。

### 2.3.2 2XX 成功

-  **200 OK** 

-  **204 No Content** ：请求已经成功处理，但是返回的响应报文不包含实体的主体部分。一般在只需要从客户端往服务器发送信息，而不需要返回数据时使用。

-  **206 Partial Content** ：表示客户端进行了范围请求。响应报文包含由 Content-Range 指定范围的实体内容。

### 2.3.3 3XX 重定向

-  **301 Moved Permanently** ：永久性重定向

-  **302 Found** ：临时性重定向

-  **303 See Other** ：和 302 有着相同的功能，但是 303 明确要求客户端应该采用 GET 方法获取资源。

- 注：虽然 HTTP 协议规定 301、302 状态下重定向时不允许把 POST 方法改成 GET 方法，但是大多数浏览器都会在 301、302 和 303 状态下的重定向把 POST 方法改成 GET 方法。

-  **304 Not Modified** ：如果请求报文首部包含一些条件，例如：If-Match，If-Modified-Since，If-None-Match，If-Range，If-Unmodified-Since，如果如果网页自请求者上次请求后再也没有更改过，则服务器会返回 304 状态码。

-  **307 Temporary Redirect** ：临时重定向，与 302 的含义类似，但是 307 要求浏览器不会把重定向请求的 POST 方法改成 GET 方法。

### 2.3.4 4XX 客户端错误

-  **400 Bad Request** ：请求报文中存在语法错误。

-  **401 Unauthorized** ：该状态码表示发送的请求需要有认证信息（BASIC 认证、DIGEST 认证）。如果之前已进行过一次请求，则表示用户认证失败。

-  **403 Forbidden** ：请求被拒绝，服务器端没有必要给出拒绝的详细理由。

-  **404 Not Found** ：请求地址不存在。

-  **407 Proxy Authentication Required** ：需要代理授权，和 401 类似，但指定请求者应当授权使用代理。

### 2.3.5 5XX 服务器错误

-  **500 Internal Server Error** ：服务器正在执行请求时发生错误。

-  **503 Service Unavailable** ：服务器暂时处于超负载或正在进行停机维护，现在无法处理请求。

## 2.4 HTTP 首部（请求头）

有 4 种类型的首部字段：通用首部字段、请求首部字段、响应首部字段和实体首部字段。

各种首部字段及其含义如下（不需要全记，仅供查阅）：

### 2.4.1 通用首部字段

| 首部字段名 | 说明 |
| :--: | :--: |
| Cache-Control | 控制缓存的行为 |
| Connection | 控制不再转发给代理的首部字段、管理持久连接|
| Date | 创建报文的日期时间 |
| Pragma | 报文指令 |
| Trailer | 报文末端的首部一览 |
| Transfer-Encoding | 指定报文主体的传输编码方式 |
| Upgrade | 升级为其他协议 |
| Via | 代理服务器的相关信息 |
| Warning | 错误通知 |

### 2.4.2 请求首部字段

| 首部字段名 | 说明 |
| :--: | :--: |
| Accept | 用户代理可处理的媒体类型 |
| Accept-Charset | 优先的字符集 |
| Accept-Encoding | 优先的内容编码 |
| Accept-Language | 优先的语言（自然语言） |
| Authorization | Web 认证信息 |
| Expect | 期待服务器的特定行为 |
| From | 用户的电子邮箱地址 |
| Host | 请求资源所在服务器 |
| If-Match | 比较实体标记（ETag） |
| If-Modified-Since | 比较资源的更新时间 |
| If-None-Match | 比较实体标记（与 If-Match 相反） |
| If-Range | 资源未更新时发送实体 Byte 的范围请求 |
| If-Unmodified-Since | 比较资源的更新时间（与 If-Modified-Since 相反） |
| Max-Forwards | 最大传输逐跳数 |
| Proxy-Authorization | 代理服务器要求客户端的认证信息 |
| Range | 实体的字节范围请求 |
| Referer | 对请求中 URI 的原始获取方 |
| TE | 传输编码的优先级 |
| User-Agent | 用户代理 |

### 2.4.3 响应首部字段

| 首部字段名 | 说明 |
| :--: | :--: |
| Accept-Ranges | 是否接受字节范围请求 |
| Age | 推算资源创建经过时间 |
| ETag | 资源的匹配信息 |
| Location | 令客户端重定向至指定 URI |
| Proxy-Authenticate | 代理服务器对客户端的认证信息 |
| Retry-After | 对再次发起请求的时机要求 |
| Server | HTTP 服务器的安装信息 |
| Vary | 代理服务器缓存的管理信息 |
| WWW-Authenticate | 服务器对客户端的认证信息 |

### 2.4.4 实体首部字段

| 首部字段名 | 说明 |
| :--: | :--: |
| Allow | 资源可支持的 HTTP 方法 |
| Content-Encoding | 实体主体适用的编码方式 |
| Content-Language | 实体主体的自然语言 |
| Content-Length | 实体主体的大小 |
| Content-Location | 替代对应资源的 URI |
| Content-MD5 | 实体主体的报文摘要 |
| Content-Range | 实体主体的位置范围 |
| Content-Type | 实体主体的媒体类型 |
| Expires | 实体主体过期的日期时间 |
| Last-Modified | 资源的最后修改日期时间 |

## 2.5 Content-Type

**（1）text/html**

Body 中返回 html 文本。

**（2）x-www-form-urlencoded**

纯文本表单的提交方式。

**（3）multitype/form-data**

含二进制文件时的提交方式。

**（4）application/json，image/jpeg,……**

单项（专项）内容提交，例如 application/json 可直接以 Body 形式传输相应 bean 类。

# 三、实际应用

## 3.1 Cookie

HTTP 协议是无状态的，主要是为了让 HTTP 协议尽可能简单。为了让它能够处理大量事务，在 HTTP/1.1 中引入 Cookie 来保存状态信息。

Cookie 是服务器发送到用户浏览器并保存在本地的一小块数据，浏览器向同一服务器再次发起请求时 Cookie 会被携带上，用于告知服务端两个请求是否来自同一浏览器。由于之后每次请求都会需要携带 Cookie 数据，因此会带来额外的性能开销（尤其是在移动环境下）。

Cookie 曾一度用于客户端数据的存储，因为当时并没有其它合适的存储办法而作为唯一的存储手段，但现在随着现代浏览器开始支持各种各样的存储方式，Cookie 渐渐被淘汰，但 Cookie 在其它用途上面仍有用武之地。

### 3.1.1 用途

- 会话状态管理（如用户登录状态、购物车、游戏分数或其它需要记录的信息）
- 个性化设置（如用户自定义设置、主题等）
- 浏览器行为跟踪（如跟踪分析用户行为等）

### 3.1.2 创建过程

服务器发送的响应报文包含 Set-Cookie 首部字段，客户端得到响应报文后把 Cookie 内容保存到浏览器（客户端）中。

```html
HTTP/1.1 200 OK
Content-type: text/html
Set-Cookie: yummy_cookie=choco
Set-Cookie: tasty_cookie=strawberry
```

客户端之后对同一个服务器发送请求时，从浏览器中读出 Cookie 信息通过添加 Cookie 请求首部字段发送给服务器。

```html
GET /sample_page.html HTTP/1.1
Host: www.example.org
Cookie: yummy_cookie=choco; tasty_cookie=strawberry
```

### 3.1.3 分类

- 会话期 Cookie：浏览器关闭之后它会被自动删除，也就是说它仅在会话期内有效。
- 持久性 Cookie：指定一个特定的过期时间（Expires）或有效期（max-age）。

```html
Set-Cookie: id=a3fWa; Expires=Wed, 21 Oct 2015 07:28:00 GMT;
```

### 3.1.4 JavaScript 获取 Cookie

通过 `Document.cookie` 属性可创建新的 Cookie，也可通过该属性访问非 HttpOnly 标记的 Cookie。

```html
document.cookie = "yummy_cookie=choco";
document.cookie = "tasty_cookie=strawberry";
console.log(document.cookie);
```

### 3.1.5 Secure 和 HttpOnly

标记为 Secure 的 Cookie 只应通过被 HTTPS 请求发送给服务端。但即便设置了 Secure 标记，敏感信息也不应该通过 Cookie 传输，因为 Cookie 有其固有的不安全性，Secure 标记也无法提供绝对的安全保障。

标记为 HttpOnly 的 Cookie 不能被 JavaScript 脚本调用。因为跨站脚本攻击 (XSS) 常常使用 JavaScript 的 `Document.cookie` API 窃取用户的 Cookie 信息，因此使用 HttpOnly 标记可以在一定程度上避免 XSS 攻击。

```html
Set-Cookie: id=a3fWa; Expires=Wed, 21 Oct 2015 07:28:00 GMT; Secure; HttpOnly
```

### 3.1.6 Session

除了可以将用户信息通过 Cookie 存储在用户浏览器中，也可以利用 Session 存储在服务器端，存储在服务器端的信息也会更加安全。

Session 可以存储在服务器上的文件、数据库或者内存中。也可以将 Session 存储在内存型数据库中，比如 Redis。

使用 Session 维护用户登录的过程如下：

- 用户进行登录时，用户提交包含用户名和密码的表单，放入 HTTP 请求报文中；
- 服务器验证该用户名和密码；
- 如果正确则把用户信息存储到 Redis 中，它在 Redis 中的 ID 称为 Session ID；
- 服务器返回的响应报文的 Set-Cookie 首部字段包含了这个 Session ID，客户端收到响应报文之后将该 Cookie 值存入浏览器中；
- 客户端之后对同一个服务器进行请求时会包含该 Cookie 值，服务器收到之后提取出 Session ID，从 Redis 中取出用户信息，继续之后的业务操作。

应该注意 Session ID 的安全性问题，不能让它被恶意攻击者轻易获取。因此 Session ID 的生成规则应当是不规律的，且还需要经常重新生成 Session ID。在对安全性要求极高的场景下，例如转账等操作，除了使用 Session 管理用户状态之外，还需要对用户进行重新验证，比如重新输入密码，或者使用短信验证码等方式。

### 3.1.7 浏览器禁用 Cookie

此时无法使用 Cookie 来保存用户信息，只能使用 Session。除此之外，不能再将 Session ID 存放到 Cookie 中，而是使用 URL 重写技术，将 Session ID 作为 URL 的参数进行传递。

### 3.1.8 Cookie 与 Session 选择

- Cookie 只能存储 ASCII 码字符串，而 Session 则可以存取任何类型的数据，因此在考虑数据复杂性时首选 Session；
- Cookie 存储在浏览器中，容易被恶意查看。如果非要将一些隐私数据存在 Cookie 中，可以将 Cookie 值进行加密，然后在服务器进行解密；
- 对于大型网站，如果用户所有的信息都存储在 Session 中，那么开销是非常大的，因此不建议将所有的用户信息都存储到 Session 中。

## 3.2 缓存

### 3.2.1 作用

- 缓解服务器压力；
- 降低客户端获取资源的延迟。

### 3.2.2 实现方法

- 让代理服务器进行缓存；
- 让客户端浏览器进行缓存。

### 3.2.3 Cache-Control

HTTP/1.1 通过 Cache-Control 首部字段来控制缓存。

**（1）禁止进行缓存** 

no-store 指令规定不能对请求或响应的任何数据进行缓存。

```html
Cache-Control: no-store
```

**（2）强制确认缓存** 

no-cache 指令规定缓存服务器需要先向源服务器验证缓存资源的有效性，只有当缓存资源有效才将能使用该缓存对客户端的请求进行响应。

```html
Cache-Control: no-cache
```

**（3）私有缓存和公共缓存** 

private 指令规定了将资源作为私有缓存，只能被单独用户所使用，一般存储在用户浏览器中。

```html
Cache-Control: private
```

public 指令规定了将资源作为公共缓存，可以被多个用户所使用，一般存储在代理服务器中。

```html
Cache-Control: public
```

**（四）缓存过期机制** 

max-age 指令出现在请求报文中，并且缓存资源的缓存时间小于该指令指定的时间，那么就能接受该缓存。

max-age 指令出现在响应报文中，表示缓存资源在缓存服务器中保存的时间。

```html
Cache-Control: max-age=31536000
```

Expires 首部字段也可以用于告知缓存服务器该资源什么时候会过期。在 HTTP/1.1 中，会优先处理 Cache-Control : max-age 指令；而在 HTTP/1.0 中，Cache-Control : max-age 指令会被忽略掉。

```html
Expires: Wed, 04 Jul 2012 08:26:05 GMT
```

### 3.2.4 缓存验证

ETag 它是资源的唯一标识。URL 不能唯一表示资源，例如 `http://www.google.com/` 有中文和英文两个资源，只有 ETag 才能对这两个资源进行唯一标识。

```html
ETag: "82e22293907ce725faf67773957acd12"
```

可以将缓存资源的 ETag 值放入 If-None-Match 首部，服务器收到该请求后，判断缓存资源的 ETag 值和资源的最新 ETag 值是否一致，如果一致则表示缓存资源有效，返回 304 Not Modified。

```html
If-None-Match: "82e22293907ce725faf67773957acd12"
```

Last-Modified 首部字段也可以用于缓存验证，它包含在源服务器发送的响应报文中，指示源服务器对资源的最后修改时间。但是它是一种弱校验器，因为只能精确到一秒，所以它通常作为 ETag 的备用方案。如果响应首部字段里含有这个信息，客户端可以在后续的请求中带上 If-Modified-Since 来验证缓存。服务器只在所请求的资源在给定的日期时间之后对内容进行过修改的情况下才会将资源返回，状态码为 200 OK。如果请求的资源从那时起未经修改，那么返回一个不带有消息主体的 304 Not Modified 响应。

```html
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
```

```html
If-Modified-Since: Wed, 21 Oct 2015 07:28:00 GMT
```

## 3.3 Authorization

第三方授权分为两种主流方式： Basic 和 Bearer 。

### 3.3.1 Basic

- 首部字段格式：Authorization: Basic Base64 (username:password)
> 例如：用户名为 passin ，密码为 123456 ，连起来则是 passin:123456 ,再对该字符串进行 Base64 编码，结果为 cGFzc2luOjEyMzQ1Ng==。即，最终添加进请求头的内容是：

```html
Authorization: Basic cGFzc2luOjEyMzQ1Ng==
```

### 3.3.2 Bearer

- 首部字段格式：Authorization: Bearer <bearer token>
- bearer token 的获取方式：通过 OAuth2 的授权流程。

#### 3.3.2.1 OAuth2 授权流程

> 1. 第三方网站向授权方网站申请第三方授权合作，目的是拿到 client id 和 client secret。
> 2. 用户在使用第三方网站时，点击授权（登陆）按钮后，跳转授权方网站，并传入 client id（第三方网站提供）作为用户的身份标识。
> 3. 授权方网站根据 client id ，将第三方网站的信息和第三方网站需要的用户权限展示给用户，并询问用户是否同意授权。
> 4. 用户同意授权后，授权方网站将页面跳转回第三方网站，并传入 Authorization code 作为用户认可的凭证。
> 5. 第三方网站将 Authorization code 发送回自己的服务器。
> 6. 服务器将 Authorization code 和自己的 client secret 一并发送给授权方的服务器，授权方服务器验证通过后，返回 access token。
> 7. 第三方网站（客户端）就可以使用 access token 作为用户授权的令牌，向授权方发送请求来获取用户信息或操作用户账号。

#### 3.3.2.2 OAuth2 微信授权登陆流程

> 1. 第三方 App 向微信开放平台申请第三方授权登陆，拿到 client id 和 client secret。
> 2. 用户在使用第三方 App 时，点击微信登陆按钮后，将通过微信 SDK 跳转至微信，并传入自己的 client id 作为用户的身份标识。
> 3. 微信通过与服务器交互，拿到第三方 App 信息，并限制在微信界面中，将第三方 App 的信息和第三方网站需要的用户权限展示给用户，并询问用户是否同意授权。
> 4. 用户点击 『同意授权』后，微信 App 和 服务区交互将同意授权的信息提交，然后跳转回第三方 App，并传回 Authorization code 作为用户认可的凭证。
> 5. 第三方 App 调用自家服务器的『微信登录』 Api，并传入 Authorization code，并等待服务器的响应。
> 6. 服务器将拿到的 Authorization code 和自己的 client secret 通过调用授权方接口发送给授权方的服务器，微信服务器验证通过后，返回 access token。
> 7. 第三方服务器拿到 access token 作为用户授权的令牌，向微信服务器请求接口来获取用户信息，微信验证同意后，返回用户信息。
> 8. 服务器拿到微信提供的用户信息后，例如该信息在自己的服务器为用户创建一个账号，并将自身服务器的用户 Id 和微信账号的用户 Id 做关联。
> 9. 用户创建完成后，向客户端返回成功的响应以及创建的用户信息。
> 10. 客户端收到响应，利用返回的用户数据进行操作，最终用户登录成功。

#### 3.3.2.3 在自家 App 中使用 Bearer token

简化掉获取 Authorization code 的过程，登录接口请求成功后，直接返回 access token。

#### 3.3.2.4 为什么引入 Authorization code

为了安全，OAuth 并不强制使用 Https，因此需要尽量保证当通信时被窃听时，依旧具有足够的安全性。
- 服务器之间的通信以及服务器自身的数据一般是较为安全的。
- 需要 Authorization code 以及第三方服务器自身才有的 client secret 才能拿到 access token。

#### 3.3.2.5 Refresh token

```
{
        "token_type" : "Bearer",
        "access_token" : "xxxxx",
        "refresh_token" : "xxxxx",
        "expires_time" : "xxxxx",
}
```

用法：accress token 只在一定时间内有效，在它失效后，可调用 refresh_token 接口，并传入 refresh_token 中的参数获取新的 access token。

目的：为了更安全。当 accress token 失窃时，窃取人只有较短时间的去利用它模拟请求接口，而 refresh_token 的值则永远存放于第三方服务器中，不容易失窃。

## 3.4 连接管理

### 3.4.1 短连接与长连接

当浏览器访问一个包含多张图片的 HTML 页面时，除了请求访问 HTML 页面资源，还会请求图片资源，如果每进行一次 HTTP 通信就要断开一次 TCP 连接，连接建立和断开的开销会很大。长连接只需要建立一次 TCP 连接就能进行多次 HTTP 通信。

从 HTTP/1.1 开始默认是长连接的，如果要断开连接，需要由客户端或者服务器端提出断开，添加头部 Connection : close；而在 HTTP/1.1 之前默认是短连接的，如果需要长连接，则添加头部 Connection : Keep-Alive。

长连接的实现方式：每间隔一定时间,使用 TCP 连接发送很短且无意义的消息,让服务器网关不将自己定义为“空闲连接”,从而防止网关关闭连接。

### 3.4.2 流水线

默认情况下，HTTP 请求是按顺序发出的，下一个请求只有在当前请求收到相应之后才会被发出。由于会受到网络延迟和带宽的限制，在下一个请求被发送到服务器之前，可能需要等待很长时间。

流水线是在同一条长连接上发出连续的请求，而不用等待响应返回，这样可以避免连接延迟。

## 3.5 内容协商

通过内容协商返回最合适的内容，例如根据浏览器的默认语言选择返回中文界面还是英文界面。

### 3.5.1 类型

**（一）服务端驱动型内容协商** 

客户端请求报文设置特定的 HTTP 首部字段，例如 Accept、Accept-Charset、Accept-Encoding、Accept-Language、Content-Languag，服务器根据这些字段返回特定的资源。

它存在以下问题：

- 服务器很难知道客户端浏览器的全部信息；
- 客户端提供的信息相当冗长（HTTP/2 协议的首部压缩机制缓解了这个问题），并且存在隐私风险（HTTP 指纹识别技术）。
- 给定的资源需要返回不同的展现形式，共享缓存的效率会降低，而服务器端的实现会越来越复杂。

**（二）代理驱动型协商** 

服务器返回 300 Multiple Choices 或者 406 Not Acceptable，客户端从中选出最合适的那个资源。

### 3.5.2 Vary

```html
Vary: Accept-Language
```

在使用内容协商的情况下，只有当缓存服务器中的缓存满足内容协商条件时，才能使用该缓存，否则应该向源服务器请求该资源。

例如，一个客户端发送了一个包含 Accept-Language 首部字段的请求之后，源服务器返回的响应包含 `Vary: Accept-Language` 内容，缓存服务器对这个响应进行缓存之后，在客户端下一次访问同一个 URL 资源，并且 Accept-Language 与缓存中的对应的值相同时才会返回该缓存。

## 3.6 内容编码

内容编码将实体主体进行压缩，从而减少传输的数据量。常用的内容编码有：gzip、compress、deflate、identity。

浏览器发送 Accept-Encoding 首部，其中包含有它所支持的压缩算法，以及各自的优先级，服务器则从中选择一种，使用该算法对响应的消息主体进行压缩，并且发送 Content-Encoding 首部来告知浏览器它选择了哪一种算法。由于该内容协商过程是基于编码类型来选择资源的展现形式的，在响应中，Vary 首部中至少要包含 Content-Encoding，这样的话，缓存服务器就可以对资源的不同展现形式进行缓存。

## 3.7 范围请求

如果网络出现中断，服务器只发送了一部分数据，范围请求可以使得客户端只请求服务器未发送的那部分数据，从而避免服务器重新发送所有数据。

### 3.7.1 Range

在请求报文中添加 Range 首部字段指定请求的范围。

```html
GET /z4d4kWk.jpg HTTP/1.1
Host: i.imgur.com
Range: bytes=0-1023
```

请求成功的话服务器返回的响应包含 206 Partial Content 状态码。

```html
HTTP/1.1 206 Partial Content
Content-Range: bytes 0-1023/146515
Content-Length: 1024
...
(binary content)
```

### 3.7.2 Accept-Ranges

响应首部字段 Accept-Ranges 用于告知客户端是否能处理范围请求，可以处理使用 bytes，否则使用 none。

```html
Accept-Ranges: bytes
```

### 3.7.3 响应状态码

- 在请求成功的情况下，服务器会返回 206 Partial Content 状态码。
- 在请求的范围越界的情况下，服务器会返回 416 Requested Range Not Satisfiable 状态码。
- 在不支持范围请求的情况下，服务器会返回 200 OK 状态码。

## 3.8 分块传输编码

分块传输编码（Chunked Transfer EnCoding） 可以把一个 Http 连接的数据分割成多块，边传输边接收。

作用：

- 服务器向客户端发送数据：让浏览器逐步显示页面，尽早给出响应，减少用户等待。
- 客户端向服务器发送数据：请求体的 Body 长度无法确定，Content-Length 不能使用。

Body 格式：

```
<length1>
<data1>
<length2>
<data2>
……
0

(最后传输 0 表示内容结束)
```

## 3.9 多部分对象集合

一份报文主体内可含有多种类型的实体同时发送，每个部分之间用 boundary 字段定义的分隔符进行分隔，每个部分都可以有首部字段。

例如，上传多个表单时可以使用如下方式：

```html
Content-Type: multipart/form-data; boundary=AaB03x

--AaB03x
Content-Disposition: form-data; name="submit-name"

--AaB03x
Content-Disposition: form-data; name="files"; filename="file1.txt"
Content-Type: text/plain

... contents of file1.txt ...
--AaB03x--
```

# 四、HTTPS

HTTP 有以下安全性问题：

- 使用明文进行通信，内容可能会被窃听；
- 不验证通信方的身份，通信方的身份有可能遭遇伪装；
- 无法证明报文的完整性，报文有可能遭篡改。

HTTPS 并不是新协议，而是让 HTTP 先和 SSL（Secure Sockets Layer）通信，再由 SSL 和 TCP 通信。也就是说 HTTPS 使用了隧道进行通信。

通过使用 SSL，HTTPS 具有了加密（防窃听）、认证（防伪装）和完整性保护（防篡改）的特性。

## 4.1 加密

加密的目的，是不希望第三者看到当前两个通讯用户的通讯内容。

### 4.1.1 对称密钥加密

对称密钥加密（Symmetric-Key Encryption），加密和解密使用同一密钥。

- 优点：运算速度快；
- 缺点：无法安全地将密钥传输给通信方。

### 4.1.2 非对称密钥加密

非对称密钥加密，又称公开密钥加密（Public-Key Encryption），加密和解密使用不同的密钥，公钥可被都可以获得，并且私钥和公钥是互相可解的。

以下用 A（客户端）和 B（服务器）描述非对称密钥加密的应用。

1. B 将公钥发送给 A；
2. A 用 B 给他的公钥加密这段消息，然后传给 B；
3. B 收到消息后使用私有密钥解密（只有接收方有私钥才能解密，从而达到保密的目的）；
4. 若 B 想向 A 回复消息，则 B 用自己的私钥加密信息，发送给 A；
5. A 收到消息后，用公钥解密信息。

- 优点：可以更安全地将公开密钥传输给通信发送方。
- 缺点：运算速度慢。

非对称密钥加密除了用来加密，还可以用来进行签名。此处以 B 给 A 回信为例说明这个过程：

1. B 对信息使用 hash 算法，生成信件摘要；
2. B 再使用其私有密钥进行加密，就生成了数字签名(signature)；
3. B 将这个签名附在要回复的信息中（例如请求头），一起发给 A；
4. A 使用公钥对数字签名进行解密，并对信息使用同样的 hash 算法，将得到 hash 值与这个签名对比是否一致来判断信息是否被篡改。

不怀好意的人也可以修改信息内容的同时也修改 hash 值，从而让它们可以相匹配，为了防止这种情况，hash 值一般都会加密后(也就是数字签名)再和信息一起发送。

## 4.2 证书（认证）

由于客户端无法确定给它公钥的就是真正的服务器，此时则需要通过使用 **证书** 来对通信方进行认证。

数字证书认证机构（CA，Certificate Authority）是客户端与服务器双方都可信赖的第三方机构。
CA 用自己的私钥对用户的身份信息(包含用户的公钥)进行数字签名，该签名和用户的身份信息一起就形成了证书。

### 4.2.1 证书包含的信息

- 签发机构的公钥：用于数字签名解密。
- 证书有效期。证书过了有效期限，证书就会作废。
- 证书的所有者（一般是某个人或者某个公司名称、机构的名称、公司网站的网址等）。
- 证书签名算法：证书签名所使用的加密算法。签发机构的公钥，根据这个算法对数字签名进行解密，得到摘要（hash 值）。
- 指纹算法：某种 hash 算法。
- 证书签名：用于保证证书的完整性，确保证书没有被修改过。其原理就是在发布证书时，发布者根据指纹算法计算证书信息得到指纹（hash 值）再使用私钥加密得到数字签名，并和证书放在一起。使用者在打开证书时，使用公钥对证书签名解密得到指纹，并使用指纹算法计算证书信息得到新的指纹，再进行对比是否一致，若一致就说明证书没有被修改过。
- 签发机构的签发机构的......(上面包含的信息)：由于签发机构可以通过另外一个更高级别的签发机构对该证书机构的公钥颁发一个证书（一般是不超过 3 级），这样形成了一个公钥证书的嵌套循环。

### 4.2.2 证书的验证过程 

- 对比证书吊销列表，检查 SSL 证书是否被证书颁发机构吊销。
- 检查 SSL 证书是否过期。
- 检查部署此 SSL 证书的网站的域名是否与证书中的域名一致。
- 检查 SSL 证书是否是由系统 **受信任的根证书颁发机构** 颁发。
- 利用签发机构的公钥对证书签名进行验证，如果验证过后所得的信息和证书签名一致，则验证通过。依次逐级对签发机构进行验证，直至本地所信任的根签发机构验证通过。

## 4.3 完整性保护

SSL 提供报文摘要功能来进行完整性保护。

HTTP 也提供了 MD5 报文摘要功能，但不是安全的。例如报文内容被篡改之后，同时重新计算 MD5 的值，通信接收方无法意识到发生了篡改。

HTTPS 的报文摘要功能之所以安全，是因为它结合了加密和认证这两个操作。因为加密之后的报文，遭到篡改之后，也很难重新计算为一致的报文摘要，因为无法轻易获取明文。

## 4.4 HTTPS 的缺点

- 因为需要进行加密解密等过程，因此速度会更慢。
- 需要支付证书授权的费用。

## 4.5 HTTPS 连接建立过程

  1. 客户端向服务器发送一条消息 **Client Hello**，Client Hello 包含的内容：客户端所能接受的（多个）SSL/TLS 版本、加密算法（对称加密算法和非对称加密算法）、hash 算法、一个客户端随机数、Server Name（表明是客户端与服务器下哪一个具体的子服务器建立链接）。
  2. 服务器接收到 Client Hello 后，将客户端随机数保存下来，并从中确定将要使用的 SSL/TLS 版本、加密算法、hash 算法。
  3. 服务器向客户端发送一条消息 **Server Hello**，消息中包含一个随机数以及确定使用的 SSL/TLS 版本、加密算法、hash 算法，客户端也将此随机数保存下来，自此客户端和服务器皆拥有一个客户端随机数和一个服务器随机数（用于计算 Pre-master Secret）。
  4. 服务器再向客户端发送证书以及选取的非对称加密算法的公钥，客户端收到证书后进行验证。验证通过后，建立信任。
  5. 客户端通过自己的信息算出一个 Pre-master Secret，并使用服务器传过来的公钥进行加密，再发送给服务器（两端皆存）。
  6. 两端利用 Pre-master Secret、客户端随机数、服务器随机数使用一个固定算法算出一个名为 Master Secret 的数据。再解析 Master Secret 得到真正进行通讯时用的客户端加密密钥、服务端加密密钥，以及用来验证身份的客户端 MAC Secret、服务器 MAC Secret。
  7. 客户端向服务器发送一条通知：“我”将使用加密通信了。
  8. 客户端再向服务器发送一条消息： **Finished**。该消息的内容为：将所有的握手信息和客户端加密密钥进行 [HMAC](https://baike.baidu.com/item/hmac/7307543?fr=aladdin) 运算的结果。
  9. 服务器向客户端发送一条消息：“我”将使用加密通信了。
  10. 服务器再向客户端发送一条消息： **Finished**。该消息的内容为：服务器也进行同样的 HMAC 运算后与客户端对比，一致则认为客户端是合法的。若合法，服务器将所有的握手信息以及客户端的 HMAC 的结果和服务器加密密钥进行 HMAC 运算后发给客户端。
  11. 客户端接收到服务端的 **Finished** 后，同样进行对比相同算法的对比验证以确定服务端以及密钥是否是合法（正确）的。若验证通过，则开始进行 HTTPS 请求。

# 五、GET 和 POST 的区别

## 5.1 作用

GET 用于获取资源，而 POST 用于传输实体主体。

## 5.2 参数

GET 和 POST 的请求都能使用额外的参数，但是 GET 的参数是拼接在 URL 中，而 POST 的参数存储在请求体中。

```
GET /test/demo_form.asp?name1=value1&name2=value2 HTTP/1.1
```

```
POST /test/demo_form.asp HTTP/1.1
Host: w3schools.com
name1=value1&name2=value2
```

不能因为 POST 参数存储在实体主体中就认为它的安全性更高，因为照样可以通过一些抓包工具（Fiddler）查看。

因为 URL 只支持 ASCII 码，因此 GET 的参数中如果存在中文等字符就需要先进行编码，例如`中文`会转换为`%E4%B8%AD%E6%96%87`，而空格会转换为`%20`。POST 支持标准字符集。

## 5.3 安全

这里所说的安全是相对的，主要指数据不会被篡改。

安全的 HTTP 方法不会改变服务器状态，也就是说它只是可读的。因此 GET 方法是安全的，而 POST 是不安全的，因为 POST 的目的是传送实体主体内容，这个内容可能是用户上传的表单数据，上传成功之后，服务器可能把这个数据存储到数据库中，因此状态也就发生了改变。

安全的方法除了 GET 之外还有：HEAD、OPTIONS。

不安全的方法除了 POST 之外还有 PUT、DELETE。

## 5.4 幂等性

幂等的 HTTP 方法，同样的请求被执行一次与连续执行多次的效果是一样的，服务器的状态也是一样的。换句话说就是，幂等方法不应该具有副作用（统计用途除外）。在正确实现的条件下，GET，HEAD，PUT 和 DELETE 等方法都是幂等的，而 POST 方法不是。所有的安全方法也都是幂等的。

GET /pageX HTTP/1.1 是幂等的。连续调用多次，客户端接收到的结果都是一样的：

```
GET /pageX HTTP/1.1
GET /pageX HTTP/1.1
GET /pageX HTTP/1.1
GET /pageX HTTP/1.1
```

POST /add_row HTTP/1.1 不是幂等的。如果调用多次，就会增加多行记录：

```
POST /add_row HTTP/1.1   -> Adds a 1nd row
POST /add_row HTTP/1.1   -> Adds a 2nd row
POST /add_row HTTP/1.1   -> Adds a 3rd row
```

DELETE /idX/delete HTTP/1.1 是幂等的，即便不同的请求接收到的状态码不一样：

```
DELETE /idX/delete HTTP/1.1   -> Returns 200 if idX exists
DELETE /idX/delete HTTP/1.1   -> Returns 404 as it just got deleted
DELETE /idX/delete HTTP/1.1   -> Returns 404
```

## 5.5 可缓存

如果要对响应进行缓存，需要满足以下条件：

- 请求报文的 HTTP 方法本身是可缓存的，包括 GET 和 HEAD，但是 PUT 和 DELETE 不可缓存，POST 在多数情况下不可缓存的。
- 响应报文的状态码是可缓存的，包括：200, 203, 204, 206, 300, 301, 404, 405, 410, 414, and 501。
- 响应报文的 Cache-Control 首部字段没有指定则不缓存。

# 参考资料

- 上野宣. 图解 HTTP[M]. 人民邮电出版社, 2014.
- [HTTP - CS-Notes](https://github.com/CyC2018/CS-Notes/blob/master/docs/notes/HTTP.md)
- [浅谈 HTTP 中 Get 与 Post 的区别 ](https://www.cnblogs.com/hyddd/archive/2009/03/31/1426026.html)
- [Cookie 与 Session 的区别 ](https://juejin.im/entry/5766c29d6be3ff006a31b84e#comment)
- [What is the difference between a URI, a URL and a URN?](https://stackoverflow.com/questions/176264/what-is-the-difference-between-a-uri-a-url-and-a-urn)

