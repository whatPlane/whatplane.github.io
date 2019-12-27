---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-gate              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
这里主要是分析一个典型的网关的流程。我们从examples目录下的main.lua文件开始.我们只关注网络部分。
```
local skynet = require "skynet"
local sprotoloader = require "sprotoloader"

local max_client = 64

skynet.start(function()
	
	local watchdog = skynet.newservice("watchdog")
	skynet.call(watchdog, "lua", "start", {//发送启动消息
		port = 8888,
		maxclient = max_client,
		nodelay = true,
	})
	skynet.error("Watchdog listen on", 8888)
	skynet.exit()
end)
```
这里启动了一个watchdog服务。watchdog启动了gate服务。watchdog.lua代码如下

```

function CMD.start(conf)//通知gate启动
	skynet.call(gate, "lua", "open" , conf)
end

skynet.start(function()
	skynet.dispatch("lua", function(session, source, cmd, subcmd, ...)
		if cmd == "socket" then
			local f = SOCKET[subcmd]
			f(...)
			-- socket api don't need return
		else
			local f = assert(CMD[cmd])//处理CMD消息
			skynet.ret(skynet.pack(f(subcmd, ...)))
		end
	end)

	gate = skynet.newservice("gate")//启动一个gate服务
end)
```

我们看gate.lua代码
```
local gateserver = require "snax.gateserver"
local handler = {}
local CMD = {}

function handler.command(cmd, source, ...)
	local f = assert(CMD[cmd])
	return f(source, ...)
end
gateserver.start(handler)//把handler和cmd里面的函数都注册到gateserver

```
gate服务的初始化主要在gateserver.start函数里面。我们看看gateserver.lua代码.
```
function gateserver.openclient(fd)
function gateserver.closeclient(fd)
function gateserver.start(handler)

```
主要就是三个函数。当然gateserver.start函数是最主要的。他注册了client协议和lua协议。消息的最终分发是通过注册的handler。关于网络协议的分发看下面的代码
```
	skynet.register_protocol {
		name = "socket",
		id = skynet.PTYPE_SOCKET,	-- PTYPE_SOCKET = 6
		unpack = function ( msg, sz )
			return netpack.filter( queue, msg, sz)//过滤数据
		end,
		dispatch = function (_, _, q, type, ...) --如果type为nil 那么表示这次收到的数据不够形成一个完整的包数据。
			queue = q
			if type then 
				MSG[type](...)//通过MSG分发网络消息
			end
		end
	}
```
lua层收到底层的网络数据后，对数据数据进行过滤，然后再分发。过滤后的数据根据不同类型进行不同的处理。
```
	function MSG.open(fd, msg)
	function MSG.close(fd)
	function MSG.error(fd, msg)
	MSG.data = dispatch_msg //收到一个完整的包
	MSG.more = dispatch_queue//收到一个完整的包+更多数据
```
因为客户端每次发送数据，格式都是 2字节+数据，而且tcp是流数据，所以我们过滤主要就是为了把接收到的数据划分为一个个完整的包提供给lua层。如果收到的数据刚好是一个完整包，那么直接通知lua，也就是通过MSG.data来处理，如果数据超过了一个完整的包，比如是大于一个完整的包，小于两个完整的包；又或者是大于三个完整的包，小于四个完整的包。这种情况就会通知MSG.more来处理。如果收到的数据不能拼凑成一个完整的包，那么不用处理，等待下次c层收到网络数据，拼凑出完整的包。过滤代码在lua_netpack.c里面。我们这里只关注收到数据的处理过程
```
static int
lfilter(lua_State *L) {
	struct skynet_socket_message *message = lua_touserdata(L,2);
	int size = luaL_checkinteger(L,3);
	char * buffer = message->buffer;
	if (buffer == NULL) {
		buffer = (char *)(message+1);
		size -= sizeof(*message);
	} else {
		size = -1;
	}

	lua_settop(L, 1);

	switch(message->type) {
	case SKYNET_SOCKET_TYPE_DATA://收到网络数据
		// ignore listen id (message->id)
		assert(size == -1);	// never padding string
		return filter_data(L, message->id, (uint8_t *)buffer, message->ud);//开始过滤数据
...
}

static inline int
filter_data(lua_State *L, int fd, uint8_t * buffer, int size) {
	int ret = filter_data_(L, fd, buffer, size);

	skynet_free(buffer);//注意最原始的网络数据在这里被释放了
	return ret;
}
```
这里主要是把网络数据取出来，进行过滤。我们的过滤操作中涉及到一个队列。这个队列主要缓存一个个完整的数据包和未完成的部分（uncomplete）。未完成部分的是指那些还需要继续接收一部分数据才能形成一个完整包的数据。下面这段代码虽然很长，但是结构很简单。主要做的事情就是，把收到的数据放入队列。然后根据接收的数据是否可以组成完整的包来通知lua
```
static int
filter_data_(lua_State *L, int fd, uint8_t * buffer, int size) {
	struct queue *q = lua_touserdata(L,1);
	struct uncomplete * uc = find_uncomplete(q, fd);
	if (uc) {
		// fill uncomplete
		if (uc->read < 0) {//第一次读取2字节的首部
			// read size
			assert(uc->read == -1);
			int pack_size = *buffer;
			pack_size |= uc->header << 8 ;//获取包的大小
			++buffer;
			--size;
			uc->pack.size = pack_size;
			uc->pack.buffer = skynet_malloc(pack_size);
			uc->read = 0;
		}
		int need = uc->pack.size - uc->read;
		if (size < need) {//数据不能组合成一个完整的包
			memcpy(uc->pack.buffer + uc->read, buffer, size);//拷贝数据到uc
			uc->read += size;
			int h = hash_fd(fd);//建立fd和uc的映射关系，方便后面查找
			uc->next = q->hash[h];
			q->hash[h] = uc;
			return 1;
		}
		memcpy(uc->pack.buffer + uc->read, buffer, need);
		buffer += need;
		size -= need;
		if (size == 0) {//因为可以组合成一个完整的包，所以通知MSG.data来处理，注意下面的类型 TYPE_DATA
			lua_pushvalue(L, lua_upvalueindex(TYPE_DATA));
			lua_pushinteger(L, fd);
			lua_pushlightuserdata(L, uc->pack.buffer);
			lua_pushinteger(L, uc->pack.size);
			skynet_free(uc);
			return 5;
		}
		// more data
		push_data(L, fd, uc->pack.buffer, uc->pack.size, 0);//把一个完整的包先压入队列缓存起来
		skynet_free(uc);
		push_more(L, fd, buffer, size);//如果还有更多的数据，那么继续处理。
		lua_pushvalue(L, lua_upvalueindex(TYPE_MORE));//注意这里是通知MSG.more来处理 TYPE_MORE
		return 2;
	} else {
		if (size == 1) {
			struct uncomplete * uc = save_uncomplete(L, fd);
			uc->read = -1;
			uc->header = *buffer;
			return 1;
		}
		int pack_size = read_size(buffer);
		buffer+=2;
		size-=2;

		if (size < pack_size) {
			struct uncomplete * uc = save_uncomplete(L, fd);
			uc->read = size;
			uc->pack.size = pack_size;
			uc->pack.buffer = skynet_malloc(pack_size);
			memcpy(uc->pack.buffer, buffer, size);
			return 1;
		}
		if (size == pack_size) {
			// just one package
			lua_pushvalue(L, lua_upvalueindex(TYPE_DATA));
			lua_pushinteger(L, fd);
			void * result = skynet_malloc(pack_size);
			memcpy(result, buffer, size);
			lua_pushlightuserdata(L, result);
			lua_pushinteger(L, size);
			return 5;
		}
		// more data
		push_data(L, fd, buffer, pack_size, 1);
		buffer += pack_size;
		size -= pack_size;
		push_more(L, fd, buffer, size);
		lua_pushvalue(L, lua_upvalueindex(TYPE_MORE));
		return 2;
	}
}
```
在这个队列数据结构中，缓存一个个完整包的是通过内部的一个数组。这是一个环形数组，如果容量不够，还会以一个固定的大小进行扩容。扩容的时机是在push一个数据后，发现容量已经不够则扩容。而缓存 uncomplete 数据，是通过一个哈希表。我们把fd值hash得到一个索引，然后把uncomplete数据加入到uncomplete链表中。我们看看这个队列的结构代码
```
struct queue {
	int cap;
	int head;
	int tail;
	struct uncomplete * hash[HASHSIZE];//缓存uncomplete数据
	struct netpack queue[QUEUESIZE];//缓存完整包
};
```
这里看一个示意图 
![](https://gitee.com/whatplane/resource/raw/master/img/ww_20190424151838.png)
- 注意：不同的fd进行hash可能会得到同一个数组的索引，所以uncomplete链表里面是存放的不同fd对应的数据

我们再来看看查找 uncomplete 的代码

```
static struct uncomplete *
find_uncomplete(struct queue *q, int fd) {
	if (q == NULL)
		return NULL;
	int h = hash_fd(fd);//获取数组的索引值
	struct uncomplete * uc = q->hash[h];
	if (uc == NULL)
		return NULL;
	if (uc->pack.id == fd) {//一次性就找到
		q->hash[h] = uc->next;
		return uc;
	}
	struct uncomplete * last = uc;
	while (last->next) {//从链表取出来。
		uc = last->next;
		if (uc->pack.id == fd) {
			last->next = uc->next;
			return uc;
		}
		last = uc;
	}
	return NULL;
}
```
再看扩容代码
```
static void
expand_queue(lua_State *L, struct queue *q) {
	//产生新的userdata，每次都是在当前的基础上，再分配QUEUESIZE来扩容 
	struct queue *nq = lua_newuserdata(L, sizeof(struct queue) + q->cap * sizeof(struct netpack));
	nq->cap = q->cap + QUEUESIZE;
	nq->head = 0;
	nq->tail = q->cap;
	memcpy(nq->hash, q->hash, sizeof(nq->hash));//把hash表的数据copy
	memset(q->hash, 0, sizeof(q->hash));
	int i;
	for (i=0;i<q->cap;i++) {//把老数据都copy到新数据里面
		int idx = (q->head + i) % q->cap;
		nq->queue[i] = q->queue[idx];
	}
	q->head = q->tail = 0;
	lua_replace(L,1);
}
```


分析完底层代码，我们看lua层是如何处理的。先看如果接收到的是一个完整的包，那么调用哪个MSG.data处理
```
local function dispatch_msg(fd, msg, sz)
		if connection[fd] then
			handler.message(fd, msg, sz)//直接通过hander处理
		else
			skynet.error(string.format("Drop message from fd (%d) : %s", fd, netpack.tostring(msg,sz)))
		end
	end

	MSG.data = dispatch_msg
```
如果是MSG.more呢
```
	local function dispatch_queue()
		local fd, msg, sz = netpack.pop(queue)//从底层队列中把缓存的完整包取出来
		if fd then
			//下面这一行代码看似有点突然，实质上说为了防止阻塞用。加入去掉这行代码，那么dispatch_msg的时候如果出现挂起，那么后面的包都无法进行分发了。
			skynet.fork(dispatch_queue)
			
			dispatch_msg(fd, msg, sz)

			for fd, msg, sz in netpack.pop, queue do
				dispatch_msg(fd, msg, sz)
			end
		end
	end

	MSG.more = dispatch_queue
```
`skynet.fork(dispatch_queue)`这行代码主要是防止阻塞的问题。因为我们的消息是在一个协程中处理的，如果在分发包的过程中，有一个包阻塞了，那么会导致后面的包都无法处理，但是我们知道skynet.lua文件中的suspend函数会在协程挂起后，处理fork_queue队列里面的协程。这样就可以继续处理后面的数据包了

#### 总结
首先是main.lua产生一个watchdog服务。watchdog产生一个gate服务。之后watchdog通知gate开始监听网络事件。当gate收到新连接时，通知watchdog，watchdog产生一个agent，同时通知agent进行一些初始化操作。agent服务会告诉gate自己的地址，这样当gate收到该连接上的数据时，直接转发到agent。下面是三个服务的关系 watchdog gate agent

![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190424174047.png)
### 结束