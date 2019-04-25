---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-lua层读数据              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---


当我们accept一个新连接后，假设我们开始读取数据，像下面这样。
```lua
skynet.start(function()
    local addr = "0.0.0.0:8001"
    skynet.error("listen " .. addr)
    local lID = socket.listen(addr)
    assert(lID)

    socket.start(lID, accept)
end)

function accept(cID, addr)
	socket.start(cID)//把新连接加入epoll
	local str = socket.read(cID)//开始读数据
end 

```
在调用socket.read函数前，我们先从底层看epoll检测到可读事件时的处理过程。
```c
socket_server_poll(struct socket_server *ss, struct socket_message * result, int * more) {
	...
	if (e->read) {
		int type;
		if (s->protocol == PROTOCOL_TCP) {
			type = forward_message_tcp(ss, s, &l, result);//可读事件发生
	...
}

static int
forward_message_tcp(struct socket_server *ss, struct socket *s, struct socket_lock *l, struct socket_message * result) {
	int sz = s->p.size;//这里每个sock默认分配的接收缓冲区是64字节
	char * buffer = MALLOC(sz);//分配内存
	int n = (int)read(s->fd, buffer, sz);//读取数据

	if (n == sz) {//根据读取数据的多少来决定是否需要对接收缓冲区大小进行调整
		s->p.size *= 2;
	} else if (sz > MIN_READ_BUFFER && n*2 < sz) {
		s->p.size /= 2;
	}

	result->opaque = s->opaque;//sock所在的服务
	result->id = s->id;//sock标识id
	result->ud = n;//我们接收数据的大小
	result->data = buffer;//这个保存我们接收的数据
	return SOCKET_DATA;//返回值 SOCKET_DATA
}

```
最后这个result把push到sock所在的的服务队列。当服务收到网络可读消息，这时候会协程来处理接收到的数据。

```lua
local function raw_dispatch_message(prototype, msg, sz, session, source)
	...
	local p = proto[prototype]
	local f = p.dispatch
	local co = co_create(f)
	session_coroutine_id[co] = session
	session_coroutine_address[co] = source
	//执行协程。注意 p.unpack(msg,sz))把数据解包
	suspend(co, coroutine_resume(co, session,source, p.unpack(msg,sz)))
	...

```

```c
static int
lunpack(lua_State *L) {//解包函数
	struct skynet_socket_message *message = lua_touserdata(L,1);
	int size = luaL_checkinteger(L,2);

	lua_pushinteger(L, message->type);
	lua_pushinteger(L, message->id);
	lua_pushinteger(L, message->ud);
	if (message->buffer == NULL) {
		lua_pushlstring(L, (char *)(message+1),size - sizeof(*message));
	} else {
		lua_pushlightuserdata(L, message->buffer);
	}

	return 4;//返回值是4个，分别是type id size data
}
```

注意下面在sock.lua中注册的分发函数
```lua
skynet.register_protocol {
	name = "socket",
	id = skynet.PTYPE_SOCKET,	-- PTYPE_SOCKET = 6
	unpack = driver.unpack,//这个函数负责把底层的网络数据解包
	dispatch = function (_, _, t, ...)//分发这个消息
		socket_message[t](...)
	end
}

-- read skynet_socket.h for these macro
-- SKYNET_SOCKET_TYPE_DATA = 1 //收到数据
socket_message[1] = function(id, size, data)
	local s = socket_pool[id]
	//把收到的数据push到socket_buffer里面
	local sz = driver.push(s.buffer, buffer_pool, data, size)
	
```
#### lua层收到数据后，怎么保存的

这里来分析`driver.push(s.buffer, buffer_pool, data, size)`是怎样把数据保存起来的。红色表示被使用。下面的图展示从一次push到第三次push。一开始分配了大量的空闲节点。我们认为是一个pool。此时push操作就是从pool中申请一块空闲节点。获得后，挂载数据，最后连接到socket_buffer的尾部。一个socket_buffer就是多个节点的链表。他的大小就是各个节点的数据大小的总和。socket_buffer是该sock收到的所有数据
![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190416165157.png)
结合代码
```c
static int
lpushbuffer(lua_State *L) {
	struct socket_buffer *sb = lua_touserdata(L,1);
	if (sb == NULL) {
		return luaL_error(L, "need buffer object at param 1");
	}
	char * msg = lua_touserdata(L,3);
	if (msg == NULL) {
		return luaL_error(L, "need message block at param 3");
	}
	int pool_index = 2;
	luaL_checktype(L,pool_index,LUA_TTABLE);
	int sz = luaL_checkinteger(L,4);
	lua_rawgeti(L,pool_index,1);
	struct buffer_node * free_node = lua_touserdata(L,-1);	// sb poolt msg size free_node
	lua_pop(L,1);
	if (free_node == NULL) {//根据是否还有空闲空间来决定是否需要扩展。初始化时16个节点
		int tsz = lua_rawlen(L,pool_index);
		if (tsz == 0)
			tsz++;
		int size = 8;
		if (tsz <= LARGE_PAGE_NODE-3) {
			size <<= tsz;
		} else {
			size <<= LARGE_PAGE_NODE-3;
		}
		lnewpool(L, size);	一次性分配size给节点
		free_node = lua_touserdata(L,-1);
		lua_rawseti(L, pool_index, tsz+1);
	}
	//每次pool[1]被使用后，新的pool[1]来源于之前pool[1]的后继节点
	lua_pushlightuserdata(L, free_node->next);	
	lua_rawseti(L, pool_index, 1);	// sb poolt msg size
	free_node->msg = msg;
	free_node->sz = sz;
	free_node->next = NULL;

	//被使用的节点都添加到 socket_buffer的尾部
	if (sb->head == NULL) {
		assert(sb->tail == NULL);
		sb->head = sb->tail = free_node;
	} else {
		sb->tail->next = free_node;
		sb->tail = free_node;
	}
	sb->size += sz;//增加sb的大小

	lua_pushinteger(L, sb->size);

	return 1;
}
```
当数据保存完之后。我们再来分析sock.read函数。
```lua
function socket.read(id, sz)
	local s = socket_pool[id]
	assert(s)
	if sz == nil then//当我们没有指定提取多少数据时，那么就提取所有数据。
		-- read some bytes
		local ret = driver.readall(s.buffer, buffer_pool)//提取所有数据
		if ret ~= "" then
			return ret
		end

		if not s.connected then
			return false, ret
		end
		assert(not s.read_required)
		s.read_required = 0
		suspend(s)//如果没有数据可以提取，那么挂起等待数据的到来
		ret = driver.readall(s.buffer, buffer_pool)//就绪读取数据
		if ret ~= "" then
			return ret
		else
			return false, ret
		end
	end
	//按照指定的大小来提取数据
	local ret = driver.pop(s.buffer, buffer_pool, sz)
	if ret then
		return ret
	end
	if not s.connected then
		return false, driver.readall(s.buffer, buffer_pool)
	end

	assert(not s.read_required)
	s.read_required = sz
	suspend(s)
	ret = driver.pop(s.buffer, buffer_pool, sz)
	if ret then
		return ret
	else
		return false, driver.readall(s.buffer, buffer_pool)
	end
end
```
上面的代码主要说明两件事。如果有数据可以提取，那么马上提取。如果没有数据可以提取，那么挂起协程等待数据到来后，再次读取。当然读取数据又分为两种方式。
- driver.readall(s.buffer, buffer_pool) 读取socket_buffer里面的所有数据
- driver.pop(s.buffer, buffer_pool, sz) 读取指定大小的数据

#### lua层怎么把网络数据提取出来
首先看提取所有数据的代码
```c
static int
lreadall(lua_State *L) {
	struct socket_buffer * sb = lua_touserdata(L, 1);
	if (sb == NULL) {
		return luaL_error(L, "Need buffer object at param 1");
	}
	luaL_checktype(L,2,LUA_TTABLE);
	luaL_Buffer b;
	luaL_buffinit(L, &b);
	while(sb->head) {//遍历整个链表，提取所有数据
		struct buffer_node *current = sb->head;
		luaL_addlstring(&b, current->msg + sb->offset, current->sz - sb->offset);
		return_free_node(L,2,sb);//回收不再使用的节点
	}
	luaL_pushresult(&b);
	sb->size = 0;
	return 1;
}

static void
return_free_node(lua_State *L, int pool, struct socket_buffer *sb) {
	struct buffer_node *free_node = sb->head;
	sb->offset = 0;
	sb->head = free_node->next;
	if (sb->head == NULL) {
		sb->tail = NULL;
	}
	//获取pool[1]节点
	lua_rawgeti(L,pool,1);
	free_node->next = lua_touserdata(L,-1);
	lua_pop(L,1);
	
	//释放不再使用的内存
	skynet_free(free_node->msg);
	free_node->msg = NULL;

	free_node->sz = 0;
	//把回收的节点作为pool[1]
	lua_pushlightuserdata(L, free_node);
	lua_rawseti(L, pool, 1);
}

```
关于回收的过程，参考下面的图示。这里原本有三个节点被使用，现在回收两个节点。每次回收的节点都是head节点，head节点回收之后就会作为pool[1]，而socket_buffer中的新head节点会由老head节点后续节点代替。

![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190416170637.png)
再看提取指定大小数据的代码.这里主要处理的问题是。指定大小的数据是从一个节点中提取，还是需要从多个节点中提取，对应的节点需要回收的问题
```c
static int
lpopbuffer(lua_State *L) {
	struct socket_buffer * sb = lua_touserdata(L, 1);
	if (sb == NULL) {
		return luaL_error(L, "Need buffer object at param 1");
	}
	luaL_checktype(L,2,LUA_TTABLE);
	int sz = luaL_checkinteger(L,3);
	if (sb->size < sz || sz == 0) {
		lua_pushnil(L);
	} else {
		pop_lstring(L,sb,sz,0);//把数据作为字符串传递给lua
		sb->size -= sz;
	}
	lua_pushinteger(L, sb->size);

	return 2;
}

static void
pop_lstring(lua_State *L, struct socket_buffer *sb, int sz, int skip) {
	struct buffer_node * current = sb->head;
	if (sz < current->sz - sb->offset) {//可以从一个节点中提取指定数据，但是该节点还会有剩余数据。
		lua_pushlstring(L, current->msg + sb->offset, sz-skip);
		sb->offset+=sz;
		return;
	}
	if (sz == current->sz - sb->offset) {//可以从该节点把数据提取完，这样也就可以收回这个节点来
		lua_pushlstring(L, current->msg + sb->offset, sz-skip);
		return_free_node(L,2,sb);
		return;
	}

	luaL_Buffer b;
	luaL_buffinit(L, &b);
	for (;;) {//需要处理多个节点才能拼凑出指定大小的数据
		int bytes = current->sz - sb->offset;
		if (bytes >= sz) {
			if (sz > skip) {
				luaL_addlstring(&b, current->msg + sb->offset, sz - skip);
			} 
			sb->offset += sz;
			if (bytes == sz) {
				return_free_node(L,2,sb);
			}
			break;
		}
		int real_sz = sz - skip;
		if (real_sz > 0) {
			luaL_addlstring(&b, current->msg + sb->offset, (real_sz < bytes) ? real_sz : bytes);
		}
		return_free_node(L,2,sb);
		sz-=bytes;
		if (sz==0)
			break;
		current = sb->head;
		assert(current);
	}
	luaL_pushresult(&b);
}


```

#### 总结
底层收到可读事件时，把数据取出，从pool中申请内存，利用socket_buffer来管理多个节点。当lua层调用read去获取数据时，如果socket_buffer有数据，则直接获取，如果没有数据，则协程进入睡眠状态，等待底层数据到来时唤醒这个协程。

### 结束