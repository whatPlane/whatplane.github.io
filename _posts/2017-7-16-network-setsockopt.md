---
layout:     post                    # 使用的布局（不需要改）
title:      setsockopt              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 网络
---
## 函数基本介绍
setsockopt主要设置sock的特性 函数原型如下

```
int setsockopt(int sock, int level, int optname, const void *optval, socklen_t optlen);
```

level指定控制套接字的层次.可以取三种值:
- **SOL_SOCKET**:通用套接字选项.
- **IPPROTO_IP**:IP选项.
- **IPPROTO_TCP**:TCP选项　

## SO_REUSERADDR
我们经常在调试服务器时设置这个选项。假设不设置，那么ctrl+c关闭服务器后，重启，就无法正常启动。因为之前的监听sock仍然处于time_wait 状态。原因是ctrl+c关闭进程后，监听sock发送fin给客户端，客户端确认并发送fin给服务端，服务端进入time_wait状态，并持续2msl，这个一般是几分钟。所以此时重启服务端，会发现端口被占用。所以我们会设置这个选项，保证及时重用端口。


```
	optlen=sizeof(option);
	option=TRUE;	
	setsockopt(serv_sock, SOL_SOCKET, SO_REUSEADDR, &option, optlen);
```
## TCP_NODELAY
在我们的游戏服务器开发中，经常会关闭这个选项。因为默认是开启nagle算法。nagel算法是为了提高TCP效率而出现的，但是游戏服务器数据交互都是很小，更在乎的是速度。所以我们会关闭这个选项。

## SO_KEEPALIVE
这个机制流程会三种结果
- 发送一个 keepalive probe，如果对端正常返回ack，那么发送端会在2小时继续发送 keepalive probe 。
- 发送一个 keepalive probe，如果对端返回rst，那么表示对端重启了，发送端关闭。
- 发送一个 keepalive probe，没有任何反应，那么会在一定间隔内继续发送 keepalive probe，直到发送次数达到了阈值就放弃，并关闭。


上面陈述中的2小时，发送间隔，发送次数都是可以设置的。但是一旦设置会影响整个主机的通讯。所以不建议这样做。
> 一般我们会在应用层来处理心跳和处理断线问题。

## SO_RCVBUF
在udp编程中，对接受缓冲区做扩大设置时，可以设置这个选项。设置的值是给系统的参考值。也就是说你设置1000，不一定接收缓冲区就直接变成了1000，系统会综合考虑。注意设置选项的顺序要尽量靠前。否则可能无效。



## 结束