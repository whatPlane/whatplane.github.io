---
layout:     post                    # 使用的布局（不需要改）
title:      udp              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 网络
---

## udp服务器
udp是一种无连接的通讯。跟我们的tcp服务器程序不同，不需要调用listen和accept函数。因为listen和accept都是在建立连接的基础上进行操作。另外发送消息和接受消息一般使用 sendto 和 recvfrom。大致跟read和write之类。
```
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
              const struct sockaddr *dest_addr, socklen_t addrlen);

ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                struct sockaddr *src_addr, socklen_t *addrlen);
```
一般情况下flags设置为0即可。
- sendto可以设置目标地址。因为不存在一对一的连接的概念，所以一个sock可以向多个sock发送消息，这样目标地址自然需要发送的时候重新设置。
- recvfrom可以获得发送者的地址。这样方便回应。

## udp客户端
- 作为客户端，一般认为是发起请求的一方。跟tcp的客户端不同，一般我们不需要调用connet去连接，而是直接调用sendto发送数据给目标。不调用bind和connect，那么什么时候分配本机的地址呢？答案是在调用sendto的时候。当然不是每次调用sendto都会自动分配一个地址，而是第一次分配后，这个地址会一直有效，直到关闭了打开的sock。

> bind当然也可以分配本机地址。只是sendto会在调用时候检查有没有分配地址，如果没有分配地址，就会主动分配。如果已经调用bind分配了，就不再分配了。

- 客户端如果每次发送给不同的服务器，那么sendto的目标地址都是不同的，这很正常。但是如果每次发送给同一个目标地址呢？可以使用connect固定目标地址。实际上这里使用connet是会提升性能的。因为每次发送数据，都会设置目标地址，发送完成后，解绑目标地址，这样会有一定的开销。而调用connet设置目标地址后，每次发送数据，就不用反复设置目标地址，解绑目标地址了。

## 边界
tcp协议是无边界的流，udp是有边界的。udp就像是邮寄信件一样（这里是为了说明他的边界性，不是说他很慢）。相反因为没有控制机制，速度会很快。在程序中体现就是，客户端发送了多少次，服务器一定会接受多少次。不想tcp协议，客户端可能发送了四五次，服务器一次read就把所有数据读取出来了。
## 注意点
因为udp少了tcp的那套重发，滑动窗口机制，流量控制，所有会出现丢包，乱序，和流量问题。流量问题主要是因为发送方发送数据太快，导致接收方来不及处理，导致接受缓冲区被后面的数据覆盖。一般解决这三个问题主要通过，设置定时，给ip包分配自定义的序列号，和扩大接受缓冲区的大小来解决。

## 结束