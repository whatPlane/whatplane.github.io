---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-lua层accept一个新连接              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---

关于怎么监听，我们已经讨论过。这次我们还是从下面这段开始分析。不过这次重点是 accept 是如何被调用的
```
local skynet    = require "skynet"
local socket    = require "skynet.socket"

skynet.start(function()
    local addr = "0.0.0.0:8001"
    skynet.error("listen " .. addr)
    local lID = socket.listen(addr)
    assert(lID)

    socket.start(lID, accept)
end)
```
当发起监听的时候，我们其实注册一个accept函数。也就是当有新的连接到来到时候，就会执行accept函数。看看注册过程
```
function socket.start(id, func)
	driver.start(id)
	return connect(id, func)
end

local function connect(id, func)
	...	
	local s = {
		id = id,
	
		callback = func,//这里我们保存了注册的连接回调函数 accept
		protocol = "TCP",
	}
	socket_pool[id] = s
	suspend(s)
	...

```
当底层收到新连接，通过epoll_wait获取新连接，代码如下 socket_server.c
```
socket_server_poll(struct socket_server *ss, struct socket_message * result, int * more) {
	...
	ss->event_n = sp_wait(ss->event_fd, ss->ev, MAX_EVENT);
	switch (s->type) {

	case SOCKET_TYPE_LISTEN: {//监听sock发现有新的连接产生
		int ok = report_accept(ss, s, result);//上报消息
		if (ok > 0) {
			return SOCKET_ACCEPT;//返回值SOCKET_ACCEPT
		} if (ok < 0 ) {
			return SOCKET_ERR;
		}
		// when ok == 0, retry
		break;
	}
	...

}

static int
report_accept(struct socket_server *ss, struct socket *s, struct socket_message *result) {
	union sockaddr_all u;
	socklen_t len = sizeof(u);
	int client_fd = accept(s->fd, &u.s, &len);//从连接完成队列中获取新的连接

	int id = reserve_id(ss);//分配一个sock内存
	if (id < 0) {
		close(client_fd);
		return 0;
	}
	socket_keepalive(client_fd);//设置保活属性
	sp_nonblocking(client_fd);//设置非阻塞
	
	//注意此时还没有把新连接加入到epoll 后续需要加入epoll才能真正开始通讯
	struct socket *ns = new_fd(ss, id, client_fd, PROTOCOL_TCP, s->opaque, false);
	if (ns == NULL) {
		close(client_fd);
		return 0;
	}
	ns->type = SOCKET_TYPE_PACCEPT;//注意状态目前是 SOCKET_TYPE_PACCEPT，也就是还没有加入epoll
	result->opaque = s->opaque;//服务id是监听sock所在的服务的id
	result->id = s->id;//监听socket的id
	result->ud = id;//新连接的id
	result->data = NULL;

	return 1;
}


```
最终会发送一个消息到监听服务所在的队列。此时lua协程主要根据下面代码执行
```
-- SKYNET_SOCKET_TYPE_ACCEPT = 4
socket_message[4] = function(id, newid, addr)
	local s = socket_pool[id]
	if s == nil then
		driver.close(newid)
		return
	end
	s.callback(newid, addr)
end
```
最终调用之前注册的accept。不过这个sock并不能马上接收数据，因为这个sock还没有加入到epoll之中。我们需要在accept里面调用 sock.start(id)。也就是把sock加入epoll。看代码
```
function socket.start(id, func)
	driver.start(id)//这里把sock加入epoll
	return connect(id, func)//最终进入挂起状态
end

```
我们再看 driver.start的底层代码最终调用（这里省略发送请求过程，这里是直接分析在ctrl_cmd里面的处理函数start_socket）
```
static int
start_socket(struct socket_server *ss, struct request_start *request, struct socket_message *result) {

	struct socket_lock l;
	socket_lock_init(s, &l);
	if (s->type == SOCKET_TYPE_PACCEPT || s->type == SOCKET_TYPE_PLISTEN) {
		//因为之前的新连接的sock的状态是 SOCKET_TYPE_PACCEPT 
		if (sp_add(ss->event_fd, s->fd, s)) {//加入epoll
		}
	
		//更新状态
		s->type = (s->type == SOCKET_TYPE_PACCEPT) ? SOCKET_TYPE_CONNECTED : SOCKET_TYPE_LISTEN;
		
		s->opaque = request->opaque;
		result->data = "start";
		
		return SOCKET_OPEN;//返回值 SOCKET_OPEN
	} else if (s->type == SOCKET_TYPE_CONNECTED) {
	

	return -1;
}
```
最后push一个消息到监听服务所在的队列。注意这个网络消息的内部类型是 SKYNET_SOCKET_TYPE_CONNECT 。之后lua层收到消息，唤醒之前的挂起的协程。执行流从 sock.start（id）后面继续。


### 结束