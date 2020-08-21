---
layout: post
category: 计算机基础
title: http基础概念
tagline: by Snail
tags: 
  - HTTP
  - 计算机
published: true
date: 2020-05-06 21:58:09 +08:00
---	
做之前爬虫的时候，一些http相关的知识有些生疏了，借着W3C上的http教程和实践来简单梳理一下。
<!--more-->

## 什么是http
http（超文本传输协议）是一种网络通信协议。它具有以下几种特点：
- 无状态：区别于TCP协议，http对于上一次请求是无记忆的
- 无链接：每次连接只处理一个请求，处理完成后即断开
- 媒体独立：任何类型的数据都可以通过http传输，客户端和服务端之间可以通过request header中的content-type来沟通

## http消息结构
请求消息包括如下4个结构：
- 请求行：包括了请求方法、URL、协议版本等信息
- 请求头部：请求头字段，常见的有cookie、accept-language、host、accept（支持的返回格式）等字段
- 空行
- 请求数据

响应消息包括了如下4个结构：
- 状态行：http协议、状态码、状态吗信息
- 消息报头：时间、返回消息内容格式（html、json等）、字符编码、内容长度
- 空行
- 响应正文

### 不常用的请求方法
常用的get，post，put，delete之外，http还支持如下几种方法：
- options：探测性请求，了解服务器性能。例子：若ajax请求是非简单请求（如跨域请求），浏览器会发送option预检。
- head：只用于获取报头，返回将只带有header信息。
- trace：用来debug，将请求的input数据返回给调用方。Jeremiah Grossman撰写的白帽报告-[关于trace可能的风险](https://www.cgisecurity.com/whitehat-mirror/WH-WhitePaper_XST_ebook.pdf)中提到了trace可能会造成安全风险，如cookie、网站证书等信息的泄漏。

### 不常见的状态码
- 1xx：服务器已收到，需要客户端继续执行操作
    - 100：继续，客户端继续其请求。
    - 101：切换协议。根据客户端请求切换到更新的协议。
    
- 3xx：重定向，需要进一步操作以完成请求
    - 301：永久移动。客户端使用新URL。
    - 302：临时移动。客户端继续使用原URL。
    - 304：Not Modified。资源未修改，不会返回任何资源。

实际开发中经常碰到的也列举如下：
- 2xx: 成功类状态
    - 200: 成功
- 4xx: 
    - 400: Bad Request，通常是客户端调用问题，如传了非法的参数
    - 401: Unauthorized，用户未验证，例如用户名和密码输错
    - 403: Forbidden，拒绝请求，资源不允许访问 
    - 404: Not Found，这个基本上人尽皆知，找不到对应的URI资源
    - 405: Method Not Allowed，http方法不允许
- 5xx: 服务端问题
    - 500: Internal Error，比如空指针
    - 502: Bad Gateway，网关问题
    - 503: Service Unavailable，可能服务正在重启导致服务不可用
    - 504: Gateway Timeout，网关问题，比如是服务端返回结果的时间超过了网关超时时长

## cookie和session
因为http无状态的特性，cookie和session被提出用来解决无状态问题。cookie简单来说是客户端（如浏览器）上存储，而session由服务端存储。
- cookie：服务器响应时使用setCookie头将cookie返回给客户端，客户端请求时将cookie带在header中传给服务端。
- session：客户端请求中带上sessionId，服务端识别sessionId并关联之前的session，否则创建一个新的sessionId并返回。

## https原理
其中非常关键的是利用了非对称加密，https的网站需要一些三方SSL证书网站认证并颁发证书，证书中包含了公钥。客户端和https服务器通信的具体步骤如下：
1. 客户端请求https服务器，服务器返回证书
2. 客户端利用证书中的公钥对自生成的随机数进行加密，并传给服务端
3. 服务端利用私钥对秘文解密，获得随机数，并用随机数对返回消息做对称加密
4. 客户端利用随机数对称解密

## http代理
http代理主要工作在[OSI模型](https://baike.baidu.com/item/OSI%E6%A8%A1%E5%9E%8B)的对话层。代理服务器主要是通过删除http请求包中的原始ip、身份信息如cookie或sessionId等方式，对服务端隐藏细节。

## restful规范
rest的英文全称是Representational State Transfer，我们可以将其视为一种web开发应用的架构风格。它最早是由Roy Fielding在2000提出的。RESTful API的设计规范主要包括如下6点：
1. Client–server：cs架构。需要遵循cs架构即客户端-服务端架构，c端使用数据，s端存储数据。方便系统的拓展和管理。
2. Stateless：无状态。每次请求都包含了所有用来理解该请求的信息，意味着服务端不存储上下文，session状态只在c端保存。
3. Cacheable：可缓存的。返回中应当标记数据是否可缓存，从而让客户端判定是否应该缓存数据以便后续重用。
4. Uniform interface：统一的接口。强调的是整体系统架构的简化和交互可见行的提升。它包括了4个接口限制：
    1.资源的标志性。RESTful中，资源体现在url中，合理的命名和区分是很重要的。
    2.通过表现来操作资源。指的是在uri中，通过特定的名词来区分对资源的不同操作（books/json、books/xml、books/csv）。或是通过http方法（post/get/delete/put）来表示具体的动作。
    3.信息自描述。选用能准确表达的名词来表示对应的资源。减少文档和过多的沟通。
    4.超媒体即是应用状态引擎。比较晦涩，参考了一篇git上的博文[jozdoo.github.io](https://jozdoo.github.io/rest/2016/09/22/REST-HATEOAS.html)。理解过来是：超媒体比如xml，用xml中字段的变化来表达出资源状态的变化。从而在不改变url的情况下，能表现出资源的变化。
5. Layered system：系统分层。系统严格分层，互相之间只通过接口交互，对于背后实现是不可见的。
6. Code on demand (optional)：按需求编码。客户端可以从服务端下载一段java applets或者scripts来拓展功能，在这一情况下，客户端并不了解如何对资源进行处理，因此客户端选择使用服务端传回的代码来处理并执行。但这个规范降低了可见性，因此它是可选的。[www.ics.uci.edu](https://www.ics.uci.edu/~fielding/pubs/dissertation/net_arch_styles.htm#sec_3_5_3)

## restful实践
- 版本号放入url中
- 用名次表示资源，而非动词
- 注意复数的使用
- 尽量将api部署在专用域名下
- 配合http方法使用
- 使用路径参数和url参数过滤信息
- 规范状态码的使用
- 准讯规范中所说的"超媒体即是应用状态引擎"（HATEOAS），在返回的超媒体中给出客户端可以参考的reference、链接等信息。如[api.github.com](api.github.com)、[api.github.com/user](api.github.com/user)

## 参考文档
- [W3CSchool-http教程](https://www.w3cschool.cn/http)
- [restfulapi.net](https://restfulapi.net/)
- [RESTful API 设计指南-阮一峰](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)