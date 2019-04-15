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
    local id  = socket.open(addr)//主动发起连接，一开始就加入了epoll，不像监听，需要sock.start来加入epoll
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

local function connect(id, func)
	...
	suspend(s)//挂起协程
	local err = s.connecting
	s.connecting = nil
	if s.connected then
		return id
	else
		socket_pool[id] = nil
		return nil, err
	end
```
发送命令给底层，底层产生一个非阻塞的sock，监听可读可写事件，调用connect函数，如果直接返回0，则连接建立，那么push一个消息给发起连接的服务的队列。如果connect没有一次性连接成功，那么通过epoll_wait发现该sock有可读或者可写事件发生时，当然需要通过getsockopt函数判断，最终发现连接成功了。此时关闭监听可写事件，push消息给发起连接的服务的队列中。lua层收到消息后，就像之前监听收到网络消息一样，最终唤醒socket.open。今天记录到这里，后面详细阐述。
未完待续....

```
static int
lconnect(lua_State *L) {

	int id = skynet_socket_connect(ctx, host, port);
	lua_pushinteger(L, id);

	return 1;
}

int 
skynet_socket_connect(struct skynet_context *ctx, const char *host, int port) {
	uint32_t source = skynet_context_handle(ctx);
	return socket_server_connect(SOCKET_SERVER, source, host, port);
}

int 
socket_server_connect(struct socket_server *ss, uintptr_t opaque, const char * addr, int port) {
	struct request_package request;
	int len = open_request(ss, &request, opaque, addr, port);
	if (len < 0)
		return -1;
	send_request(ss, &request, 'O', sizeof(request.u.open) + len);//发送请求
	return request.u.open.id;
}

```
最终在 ctrl_cmd函数里面处理 `O`类型的请求。也就是调用 open_socket
```
open_socket(struct socket_server *ss, struct request_open * request, struct socket_message *result) {
	...
	sock = socket( ai_ptr->ai_family, ai_ptr->ai_socktype, ai_ptr->ai_protocol );//产生一个sock文件
	socket_keepalive(sock);//设置保活属性
	sp_nonblocking(sock);//设置为非阻塞
	status = connect( sock, ai_ptr->ai_addr, ai_ptr->ai_addrlen);//发起主动连接
	ns = new_fd(ss, id, sock, PROTOCOL_TCP, request->opaque, true);//分配一个sock，同时加入epoll监听

	...
	if(status == 0) {
		ns->type = SOCKET_TYPE_CONNECTED;//已经连接成功
		struct sockaddr * addr = ai_ptr->ai_addr;
		void * sin_addr = (ai_ptr->ai_family == AF_INET) ? (void*)&((struct sockaddr_in *)addr)->sin_addr : (void*)&((struct sockaddr_in6 *)addr)->sin6_addr;
		if (inet_ntop(ai_ptr->ai_family, sin_addr, ss->buffer, sizeof(ss->buffer))) {
			result->data = ss->buffer;
		}
		freeaddrinfo( ai_list );
		//注意 如果一次性连接成功 ，那么直接返回SOCKET_OPEN，这时会往发起连接的服务push一个消息。
		return SOCKET_OPEN;
	} else {
		ns->type = SOCKET_TYPE_CONNECTING;//还在连接中，需要监听是否连接成功
		sp_write(ss->event_fd, ns->fd, ns, true);//监听可写事件
	}

	return -1
}

static struct socket *
new_fd(struct socket_server *ss, int id, int fd, int protocol, uintptr_t opaque, bool add) {
	struct socket * s = &ss->slot[HASH_ID(id)];//分配一个sock
	assert(s->type == SOCKET_TYPE_RESERVE);

	if (add) {
		if (sp_add(ss->event_fd, fd, s)) {//加入epoll监听，监听可读事件
			s->type = SOCKET_TYPE_INVALID;
			return NULL;
		}
	}
	//进行各种初始化
	s->id = id;
	s->fd = fd;
	s->sending = ID_TAG16(id) << 16 | 0;
	s->protocol = protocol;
	s->p.size = MIN_READ_BUFFER;
	s->opaque = opaque;
	s->wb_size = 0;
	s->warn_size = 0;
	check_wb_list(&s->high);
	check_wb_list(&s->low);
	spinlock_init(&s->dw_lock);
	s->dw_buffer = NULL;
	s->dw_size = 0;
	return s;
}
```
之后通过epoll_wait去检查连接是否成功。注意异步connect连接成功的判断方法和后续设置
```
socket_server_poll(struct socket_server *ss, struct socket_message * result, int * more) {
			...
			ss->event_n = sp_wait(ss->event_fd, ss->ev, MAX_EVENT);//检测事件的发生
			...
			switch (s->type) {
			case SOCKET_TYPE_CONNECTING://当前sock是正在连接状态
				return report_connect(ss, s, &l, result);
}

static int
report_connect(struct socket_server *ss, struct socket *s, struct socket_lock *l, struct socket_message *result) {
	int error;
	socklen_t len = sizeof(error);  
	int code = getsockopt(s->fd, SOL_SOCKET, SO_ERROR, &error, &len);//获取错误码  
	if (code < 0 || error) {  
		return SOCKET_ERR;
	} else {//已经连接成功
		s->type = SOCKET_TYPE_CONNECTED;
		result->opaque = s->opaque;
		result->id = s->id;
		result->ud = 0;
		if (nomore_sending_data(s)) {
			sp_write(ss->event_fd, s->fd, s, false);//关闭监听可写事件
		}
	
		result->data = NULL;
		return SOCKET_OPEN;//返回值 push一个消息到发起连接的服务队列，
	}
}


```
这个时候，lua层收到底层的连接成功的消息。在sock.lua中处理。接下来的步骤跟 lua层发起监听，之后收到底层反馈的监听成功消息处理流程一样。具体看 [skynet-lua层网络监听过程](https://whatplane.github.io/2018/03/17/skynet-lua%E5%B1%82%E7%BD%91%E7%BB%9C%E7%9B%91%E5%90%AC%E8%BF%87%E7%A8%8B/) 即先把这个睡眠的协程加入到一个 唤醒队列。之后suspend处理唤醒队列里面的协程。这时候，回到本片文章最开始的，也就是挂起的地方开始执行。
```
-- SKYNET_SOCKET_TYPE_CONNECT = 2
socket_message[2] = function(id, _ , addr)
	local s = socket_pool[id]
	if s == nil then
		return
	end
	-- log remote addr
	s.connected = true
	wakeup(s)
end

```
注意 listen和connect 成功后，网络消息内部的类型都是 SKYNET_SOCKET_TYPE_CONNECT 。

### 结束