---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-epoll网络监听              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
#### 网络处理初始化 
网络开启 skynet_socket_init .skynet_socket内部是使用 socket_server。初始化网络主要做了下面几件事
- 产生一个epoll例程
- 产生一对管道描述符。分别是写和读。同时把读描述符加入epoll
- 初始化 socket_server 结构体，我们叫他网络总管吧。内部有一个数组slot，里面是大量的预分配的socket结构体，用于将来处理每一个连接用。
- 每个socket 结构体。主要成员有 在slot的槽位id 和 发送数据缓冲区。  

注意下面针对结构体的注释

```
struct socket {
	uintptr_t opaque;	对应服务的handle
	struct wb_list high; 高优先级发送数据链表
	struct wb_list low;  低优先级发送数据链表
	int64_t wb_size;
	uint32_t sending;
	int fd;
	int id; 在slot里面的槽位
	uint8_t protocol;
	uint8_t type; 当前socket的一个类型记录
	uint16_t udpconnecting;
	int64_t warn_size;
	union {
		int size;
		uint8_t udp_address[UDP_ADDRESS_SIZE];
	} p;
	struct spinlock dw_lock;
	int dw_offset;
	const void * dw_buffer; 直接写数据buffer
	size_t dw_size;
};


struct socket_server {
	int recvctrl_fd;
	int sendctrl_fd;
	int checkctrl;
	poll_fd event_fd;
	int alloc_id; 当前已经被使用的 socket 计数
	int event_n;
	int event_index;
	struct socket_object_interface soi;
	struct event ev[MAX_EVENT];
	struct socket slot[MAX_SOCKET]; slot数组
	char buffer[MAX_INFO];
	uint8_t udpbuffer[MAX_UDP_PACKAGE];
	fd_set rfds;
};
```

这里是初始化调用的函数代码
```
skynet_socket_init() {
	SOCKET_SERVER = socket_server_create();
}

socket_server_create() {
int i;
	int fd[2];
	poll_fd efd = sp_create();//产生epoll例程

	if (pipe(fd)) {//产生管道
	
	}
	if (sp_add(efd, fd[0], NULL)) {//把管道的读端加入epoll
	
	}

	struct socket_server *ss = MALLOC(sizeof(*ss));//分配大量堆空间
	ss->event_fd = efd;
	ss->recvctrl_fd = fd[0];
	ss->sendctrl_fd = fd[1];
	ss->checkctrl = 1;

	for (i=0;i<MAX_SOCKET;i++) { //提前分配大量的socket
		struct socket *s = &ss->slot[i];
		s->type = SOCKET_TYPE_INVALID; //类型都设置为无效
		clear_wb_list(&s->high);
		clear_wb_list(&s->low);
	}
	ss->alloc_id = 0; //设置已经被占用的socket计数
	ss->event_n = 0;
	ss->event_index = 0;
	memset(&ss->soi, 0, sizeof(ss->soi));
	FD_ZERO(&ss->rfds);
	assert(ss->recvctrl_fd < FD_SETSIZE);

	return ss;
}

```

#### 网络处理线程
```
static void *
thread_socket(void *p) {
	struct monitor * m = p;
	skynet_initthread(THREAD_SOCKET);
	for (;;) {
		int r = skynet_socket_poll();//这里不断的处理epoll事件
		if (r==0)
			break;
		if (r<0) {
			CHECK_ABORT
			continue;
		}
		wakeup(m,0);
	}
	return NULL;
}


```

我们从lua层监听分析起。lua层调用 `local lID = socket.listen(addr)`对应的底层c函数的位置在 lua-socket.c文件。 addr = "0.0.0.0:8001"
```
static int
llisten(lua_State *L) {
	const char * host = luaL_checkstring(L,1);
	int port = luaL_checkinteger(L,2);
	int backlog = luaL_optinteger(L,3,BACKLOG);
	struct skynet_context * ctx = lua_touserdata(L, lua_upvalueindex(1));//这里获得了发起监听的服务
	int id = skynet_socket_listen(ctx, host,port,backlog);//这里开始调用监听
	if (id < 0) {
		return luaL_error(L, "Listen error");
	}

	//这里的id不是sock的文件描述符，可以认为是 socket 在 SOCKET_SERVER的slot槽位 
	lua_pushinteger(L,id);
	return 1;
}

skynet_socket_listen(struct skynet_context *ctx, const char *host, int port, int backlog) {
	uint32_t source = skynet_context_handle(ctx);//保存了发起监听的服务
	return socket_server_listen(SOCKET_SERVER, source, host, port, backlog);
}

```

上面的 SOCKET_SERVER 就是我们的网络处理总管。
```
socket_server_listen(struct socket_server *ss, uintptr_t opaque, const char * addr, int port, int backlog) {
	int fd = do_listen(addr, port, backlog);//这里内部调用了传统网络编程的 listen bind 
	if (fd < 0) {
		return -1;
	}
	struct request_package request;
	int id = reserve_id(ss);//从网络总管 SOCKET_SERVER 那里分配一个槽位，可以索引一个自定义的socket结构体
	if (id < 0) {
		close(fd);
		return id;
	}
	request.u.listen.opaque = opaque;//发起监听的 服务
	request.u.listen.id = id;// SOCKET_SERVER 分配的槽位标志
	request.u.listen.fd = fd;// 监听sock的文件描述符
	send_request(ss, &request, 'L', sizeof(request.u.listen)); //通过管道发起请求
	return id;
}
```

下面看 send_request 。这要通过管道的写端发送`L`请求，也就是发送监听请求。
```
struct request_package {
	uint8_t header[8];	// 6 bytes dummy 暂时只使用后面两个字节
	union {
		char buffer[256];
		struct request_open open;
		struct request_send send;
		struct request_send_udp send_udp;
		struct request_close close;
		struct request_listen listen;//监听请求
		struct request_bind bind;
		struct request_start start;
		struct request_setopt setopt;
		struct request_udp udp;
		struct request_setudp set_udp;
	} u;
	uint8_t dummy[256];
};

static void
send_request(struct socket_server *ss, struct request_package *request, char type, int len) {
	request->header[6] = (uint8_t)type;
	request->header[7] = (uint8_t)len;
	for (;;) {//发送的字节总数是 len+2，起始位置是header[6]
		ssize_t n = write(ss->sendctrl_fd, &request->header[6], len+2);
		if (n<0) {
			if (errno != EINTR) {
				fprintf(stderr, "socket-server : send ctrl command error %s.\n", strerror(errno));
			}
			continue;
		}
		assert(n == len+2);
		return;
	}
}

```

下面是处理请求的代码
```
socket_server_poll(struct socket_server *ss, struct socket_message * result, int * more) {
	for (;;) {
		if (ss->checkctrl) {
			if (has_cmd(ss)) {//这里通过 select发现是否有所谓的命令请求
				int type = ctrl_cmd(ss, result);//处理请求。
				if (type != -1) { //一般socket.start命令的处理结果是 SOCKET_OPEN  
					clear_closed_event(ss, result, type);
					return type;
				} else
					continue;//socket.listen命令的处理结果是 -1，也就是ctrl_cmd 的结果是 -1 
			} else {
				ss->checkctrl = 0;
			}
		}

static int
has_cmd(struct socket_server *ss) {
	struct timeval tv = {0,0};//设置为非阻塞立即返回检查模式
	int retval;

	FD_SET(ss->recvctrl_fd, &ss->rfds);监听可读事件

	retval = select(ss->recvctrl_fd+1, &ss->rfds, NULL, NULL, &tv);
	if (retval == 1) {//说明有可读事件发生
		return 1;
	}
	return 0;
}

static int
ctrl_cmd(struct socket_server *ss, struct socket_message *result) {
	int fd = ss->recvctrl_fd;

	uint8_t buffer[256];
	uint8_t header[2];
	block_readpipe(fd, header, sizeof(header));
	int type = header[0];//获得请求的类型
	int len = header[1];//获得请求数据的长度
	block_readpipe(fd, buffer, len);//提取请求数据

	switch (type) {
	
	case 'L'://处理监听请求
		return listen_socket(ss,(struct request_listen *)buffer, result);
｝


static int
listen_socket(struct socket_server *ss, struct request_listen * request, struct socket_message *result) {
	int id = request->id;
	int listen_fd = request->fd;
	//主要是初始化 socket结构体 ，下面代码的最后一个参数 false表示不加入epoll
	struct socket *s = new_fd(ss, id, listen_fd, PROTOCOL_TCP, request->opaque, false);

	//设置 类型为 SOCKET_TYPE_PLISTEN ，这里还没有加入到 epoll
	s->type = SOCKET_TYPE_PLISTEN;
	return -1;//注意这里的返回值


```

到这里 我们在lua层调用的` socket.listen(addr)`还并没有加入epoll，这个时候socket的type还是SOCKET_TYPE_PLISTEN，只有执行完 `socket.start(lID, accept)`，才算真正开始监听，这个时候 socket的type是SOCKET_TYPE_LISTEN。socket.start跟socket.listen类似，都是通过管道发送请求，然后在网络的主循环时被优先处理。
这里直接展示最后处理这个命令的代码

```
static int
start_socket(struct socket_server *ss, struct request_start *request, struct socket_message *result) {
	int id = request->id;
	result->id = id;
	result->opaque = request->opaque;
	result->ud = 0;
	result->data = NULL;
	struct socket *s = &ss->slot[HASH_ID(id)];//通过之前socket.listen(addr)返回的id找出 socket
	if (s->type == SOCKET_TYPE_INVALID || s->id !=id) {
		result->data = "invalid socket";
		return SOCKET_ERR;
	}
	struct socket_lock l;
	socket_lock_init(s, &l);
	if (s->type == SOCKET_TYPE_PACCEPT || s->type == SOCKET_TYPE_PLISTEN) {
		if (sp_add(ss->event_fd, s->fd, s)) {//把fd加入epoll，进行监听
			force_close(ss, s, &l, result);
			result->data = strerror(errno);
			return SOCKET_ERR;
		}
	
		//这里说明主动监听和被动连接最终要完成 都需要调用 start_socket 函数。
		s->type = (s->type == SOCKET_TYPE_PACCEPT) ? SOCKET_TYPE_CONNECTED : SOCKET_TYPE_LISTEN;
		s->opaque = request->opaque;
		result->data = "start";
		return SOCKET_OPEN;//最后的返回值 SOCKET_OPEN
	} else if (s->type == SOCKET_TYPE_CONNECTED) {
		// todo: maybe we should send a message SOCKET_TRANSFER to s->opaque
		s->opaque = request->opaque;
		result->data = "transfer";
		return SOCKET_OPEN;
	}
	// if s->type == SOCKET_TYPE_HALFCLOSE , SOCKET_CLOSE message will send later
	return -1;
}
```
这个返回值 SOCKET_OPEN 就是 socket_server_poll 的返回值。回到最开始网络线程循环处理的地方 skynet_socket_poll

```
int 
skynet_socket_poll() {
	struct socket_server *ss = SOCKET_SERVER;
	assert(ss);
	struct socket_message result;
	int more = 1;
	int type = socket_server_poll(ss, &result, &more);//返回值是 SOCKET_OPEN
	switch (type) {
	case SOCKET_EXIT:
		return 0;
	case SOCKET_DATA:
		forward_message(SKYNET_SOCKET_TYPE_DATA, false, &result);
		break;
	case SOCKET_CLOSE:
		forward_message(SKYNET_SOCKET_TYPE_CLOSE, false, &result);
		break;
	case SOCKET_OPEN://这里我们把结果转发到对应的发起监听的服务
		forward_message(SKYNET_SOCKET_TYPE_CONNECT, true, &result);
		break;
	
	default:
		skynet_error(NULL, "Unknown socket message type %d.",type);
		return -1;
	}
	if (more) {
		return -1;
	}
	return 1;
}
```

转发的代码如下。这里是把 socket_message 消息 转变成 skynet_socket_message 消息，再把 skynet_socket_message 当作是skynet_message 消息的数据部分。也就是把所谓的网络消息转变本地消息。最后把消息push到发起监听的服务的队列中
```
static void
forward_message(int type, bool padding, struct socket_message * result) {
	struct skynet_socket_message *sm;
	size_t sz = sizeof(*sm);
	if (padding) {
		if (result->data) {
			size_t msg_sz = strlen(result->data);
			if (msg_sz > 128) {
				msg_sz = 128;
			}
			sz += msg_sz;
		} else {
			result->data = "";
		}
	}
	sm = (struct skynet_socket_message *)skynet_malloc(sz);//skynet网络消息
	sm->type = type;//此时是 SKYNET_SOCKET_TYPE_CONNECT
	sm->id = result->id;//ss 中 socket的位置 
	sm->ud = result->ud;//0
	if (padding) {
		sm->buffer = NULL;
		memcpy(sm+1, result->data, sz - sizeof(*sm));
	} else {
		sm->buffer = result->data;
	}

	struct skynet_message message;//节点内部消息
	message.source = 0;//注意这里都默认是0
	message.session = 0;
	message.data = sm;
	message.sz = sz | ((size_t)PTYPE_SOCKET << MESSAGE_TYPE_SHIFT);//高八位 设置类型为 PTYPE_SOCKET
	
	if (skynet_context_push((uint32_t)result->opaque, &message)) {//把消息push到对应服务的队列
		// todo: report somewhere to close socket
		// don't call skynet_socket_close here (It will block mainloop)
		skynet_free(sm->buffer);
		skynet_free(sm);
	}
```
这里主要是讨论了 lua曾调用socket.listen 和socket.start底层做到事情，接下一篇讨论 [lua层网络监听过程](https://whatplane.github.io/tags/#skynet%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)
### 结束