---
layout:     post                    # 使用的布局（不需要改）
title:      http的基本原理              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---

http主要用于内部网络通讯。比如说两个skynet节点或者是提供简单的web服务给后台服务。
#### 基本原理
http是基于tcp的应用层协议。基本流程是：
- 客户端发起建立连接的请求
- 客户端和服务器建立连接
- 客户端发送请求方法 类似 GET POST
- 服务器响应客户端
- 服务器关闭连接

也就是每一次发起http请求都是走这套流程。在服务器看来，同一个主机连续发送多次请求，他也认为是完全独立的请求。
#### 客户端请求格式
![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190626104117-min.png)
![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190626103548.png)
看具体的例子 **注意空格** 当然这里换行和返回字符没有体现出来。也就是`\r\n`。消息头，也就是那些键值对，主要是告诉服务器，当前客户端的相关信息，比如`Accept: image/gif, image/jpeg`表示当前客户端可以解析的图片格式。

```
GET /index.html HTTP/1.1
User-Agent: curl/7.16.3 libcurl/7.16.3 OpenSSL/0.9.7l zlib/1.2.3
Accept: image/gif, image/jpeg
Accept-Language: en, mi

```

#### 服务器响应的格式
![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190626104053-min.png)
同样服务器返回的消息头，也就是那些键值对，主要是告诉客户端服务器这次发送数据的相关情况。比如`Content-Length: 2048`表示当前数据的长度大小。大概是下面这样子

```
HTTP/1.1 200 OK
Date: Mon, 27 Jul 2009 12:28:53 GMT
Server: SimpleWebServer
Content-Length: 50 
Content-Type: text/html

<html>
<head><title>这是网页的标题</title></head>
<body><font size=+5>
hello web<br>
</font></body>
</html>
```
#### skynet提供的http服务器
点击进入[skynet中提供的web服务]()
### 结束