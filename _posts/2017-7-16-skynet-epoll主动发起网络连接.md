---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-epoll主动发起连接              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
作为服务器，我们也可能需要作为客户端来连接其他服务器，那么主动发起连接不可避免。我们根据下面这段代码展开
```
skynet.start(function()
    local addr = "127.0.0.1:8001"
    skynet.error("connect ".. addr)
    local id  = socket.open(addr)
    assert(id)
    --启动读协程
    skynet.fork(client, id)
end)
```
我这里主要讨论socket.open(addr)的过程。
```
function socket.open(addr, port)
	local id = driver.connect(addr,port)//发送创建新连接的命令给底层
	return connect(id)//进入睡眠状态
end
```
发送命令给底层，底层产生一个非阻塞的sock，监听可读可写事件，调用connect函数，如果直接返回0，则连接建立，那么push一个消息给发起连接的服务的队列。如果connect没有一次性连接成功，那么通过epoll_wait发现该sock有可读或者可写事件发生时，当然需要通过getsockopt函数判断，最终发现连接成功了。此时关闭监听可写事件，push消息给发起连接的服务的队列中。lua层收到消息后，就像之前监听收到网络消息一样，最终唤醒socket.open。今天记录到这里，后面详细阐述。
未完待续....


### 结束