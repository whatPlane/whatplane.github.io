---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-skynet.pack skynet.unpack              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---

### skynet.pack
当我们要给另外一个服务发送数据时，我们一般先打包数据。
```
skynet.pack(...)
```
这个函数，主要是把lua层数据转化成底层数据，最后返回给lua层一个 lightuserdata + size 。具体细节上这样
- 不断的取出参数，根据大小分配内存，再加入链表
- 最后根据所有链表的长度分配一个大内存块，把所有链表的数据copy进去
- 释放链表占用的内存
- 返回给lua层 大内存块的指针 + 大内存块的大小


看代码
```
LUAMOD_API int
luaseri_pack(lua_State *L) {
	struct block temp;
	temp.next = NULL;
	struct write_block wb;
	wb_init(&wb, &temp);
	pack_from(L,&wb,0);//把所有参数加入链表
	assert(wb.head == &temp);
	seri(L, &temp, wb.len);//把链表数据copy到大内存块

	wb_free(&wb);//释放链表的内存

	return 2;
}

static void
wb_init(struct write_block *wb , struct block *b) {
	wb->head = b;
	assert(b->next == NULL);
	wb->len = 0;//链表的总大小
	wb->current = wb->head;//当前节点
	wb->ptr = 0;//当前节点已经使用的空间大小
}
```
这里看看 pack_from 代码
```
static void
pack_from(lua_State *L, struct write_block *b, int from) {
	int n = lua_gettop(L) - from;
	int i;
	for (i=1;i<=n;i++) {
		pack_one(L, b , from + i, 0);//把每一个参数加入链表
	}
}


static void
pack_one(lua_State *L, struct write_block *b, int index, int depth) {
	if (depth > MAX_DEPTH) {
		wb_free(b);
		luaL_error(L, "serialize can't pack too depth table");
	}
	int type = lua_type(L,index);
	switch(type) {
	case LUA_TNIL://nil类型
		wb_nil(b);
		break;
	case LUA_TNUMBER: {//数字类型
		if (lua_isinteger(L, index)) {
			lua_Integer x = lua_tointeger(L,index);
			wb_integer(b, x);
		} else {
			lua_Number n = lua_tonumber(L,index);
			wb_real(b,n);
		}
		break;
	}
	case LUA_TBOOLEAN: //布尔类型
		wb_boolean(b, lua_toboolean(L,index));
		break;
	case LUA_TSTRING: {
		size_t sz = 0;
		const char *str = lua_tolstring(L,index,&sz);
		wb_string(b, str, (int)sz);
		break;
	}
	case LUA_TLIGHTUSERDATA://
		wb_pointer(b, lua_touserdata(L,index));
		break;
	case LUA_TTABLE: {//表类型
		if (index < 0) {
			index = lua_gettop(L) + index + 1;
		}
		wb_table(L, b, index, depth+1);
		break;
	}
	default:
		wb_free(b);
		luaL_error(L, "Unsupport type %s to serialize", lua_typename(L, type));
	}
}

```
#### 数字类型
下面是数字类型加入链表的过程。有两点需要注意
- 根据数字类型占用的字节数，来具体分配内存。
- 加入前首先添加类型相关的数据，这个占用固定的一个字节


```
#define COMBINE_TYPE(t,v) ((t) | (v) << 3)


static inline void
wb_integer(struct write_block *wb, lua_Integer v) {
	int type = TYPE_NUMBER;
	if (v == 0) {
		uint8_t n = COMBINE_TYPE(type , TYPE_NUMBER_ZERO);//把主类型和次类型合并为一个字节
		wb_push(wb, &n, 1);//加入到链表
	} else if (v != (int32_t)v) {
		uint8_t n = COMBINE_TYPE(type , TYPE_NUMBER_QWORD);//TYPE_NUMBER_QWORD
		int64_t v64 = v;
		wb_push(wb, &n, 1);//添加类型
		wb_push(wb, &v64, sizeof(v64));//添加实际数据
	} else if (v < 0) {
		int32_t v32 = (int32_t)v;
		uint8_t n = COMBINE_TYPE(type , TYPE_NUMBER_DWORD);//TYPE_NUMBER_DWORD
		wb_push(wb, &n, 1);
		wb_push(wb, &v32, sizeof(v32));
	} else if (v<0x100) {
		uint8_t n = COMBINE_TYPE(type , TYPE_NUMBER_BYTE);//TYPE_NUMBER_BYTE
		wb_push(wb, &n, 1);
		uint8_t byte = (uint8_t)v;
		wb_push(wb, &byte, sizeof(byte));
	} else if (v<0x10000) {
		uint8_t n = COMBINE_TYPE(type , TYPE_NUMBER_WORD);//TYPE_NUMBER_WORD
		wb_push(wb, &n, 1);
		uint16_t word = (uint16_t)v;
		wb_push(wb, &word, sizeof(word));
	} else {
		uint8_t n = COMBINE_TYPE(type , TYPE_NUMBER_DWORD);
		wb_push(wb, &n, 1);
		uint32_t v32 = (uint32_t)v;
		wb_push(wb, &v32, sizeof(v32));
	}
}

static inline void
wb_real(struct write_block *wb, double v) {
	uint8_t n = COMBINE_TYPE(TYPE_NUMBER , TYPE_NUMBER_REAL);//
	wb_push(wb, &n, 1);
	wb_push(wb, &v, sizeof(v));
}
```
最后看看怎么加入到 wb_push.链表中每个节点的大小最大是 BLOCK_SIZE。
```
inline static void
wb_push(struct write_block *b, const void *buf, int sz) {
	const char * buffer = buf;
	if (b->ptr == BLOCK_SIZE) {//需要分配新的节点来
_again:
		b->current = b->current->next = blk_alloc();
		b->ptr = 0;
	}
	if (b->ptr <= BLOCK_SIZE - sz) {//完全可以容纳新加入到数据
		memcpy(b->current->buffer + b->ptr, buffer, sz);
		b->ptr+=sz;
		b->len+=sz;
	} else {//只能容纳部分新增数据，需要分配新节点了
		int copy = BLOCK_SIZE - b->ptr;
		memcpy(b->current->buffer + b->ptr, buffer, copy);
		buffer += copy;
		b->len += copy;
		sz -= copy;
		goto _again;
	}
}
```

#### lightuserdata
看看 lightuserdata是如何加入链表的
```
case LUA_TLIGHTUSERDATA:
		wb_pointer(b, lua_touserdata(L,index));//取出数据
		break;

}

static inline void
wb_pointer(struct write_block *wb, void *v) {
	uint8_t n = TYPE_USERDATA;//设置类型
	wb_push(wb, &n, 1);//添加类型
	wb_push(wb, &v, sizeof(v));//添加实际数据
}
```

### skynet.unpack
上面解释的打包过程，很明显，解包就是相反的过程。skynet.unpack的参数有两个，就是 指针和大小 。而每一个数据本质上是 一个字节+value构成。那一个首字节代表了类型信息，value是真正的数据 ，我们看底层实现
```
int
luaseri_unpack(lua_State *L) {
	if (lua_isnoneornil(L,1)) {
		return 0;
	}
	void * buffer;
	int len;
	if (lua_type(L,1) == LUA_TSTRING) {
		size_t sz;
		 buffer = (void *)lua_tolstring(L,1,&sz);
		len = (int)sz;
	} else {
		buffer = lua_touserdata(L,1);//指针
		len = luaL_checkinteger(L,2);//大小
	}

	lua_settop(L,1);
	struct read_block rb;
	rball_init(&rb, buffer, len);//初始化

	int i;
	for (i=0;;i++) {//不断的把数据单元取出来
		if (i%8==7) {
			luaL_checkstack(L,LUA_MINSTACK,NULL);
		}
		uint8_t type = 0;
		uint8_t *t = rb_read(&rb, sizeof(type));//取出类型
		if (t==NULL)
			break;
		type = *t;
		push_value(L, &rb, type & 0x7, type>>3);//根据类型取出数据，然后压入lua栈中
	}

	// Need not free buffer

	return lua_gettop(L) - 1;
}
```
我们看push_value函数。主要是根据不同类型取出数据。
```
static void
push_value(lua_State *L, struct read_block *rb, int type, int cookie) {
	switch(type) {
	case TYPE_NIL:
		lua_pushnil(L);
		break;
	case TYPE_BOOLEAN:
		lua_pushboolean(L,cookie);
		break;
	case TYPE_NUMBER:
		if (cookie == TYPE_NUMBER_REAL) {
			lua_pushnumber(L,get_real(L,rb));//取出数字类型数据
		} else {
			lua_pushinteger(L, get_integer(L, rb, cookie));
		}
		break;
	case TYPE_USERDATA:
		lua_pushlightuserdata(L,get_pointer(L,rb));//获取userdata对应的指针
		break;
......
}

static double
get_real(lua_State *L, struct read_block *rb) {
	double n;
	double * pn = rb_read(rb,sizeof(n));//读取数据
	if (pn == NULL)
		invalid_stream(L,rb);
	memcpy(&n, pn, sizeof(n));//把数据copy一份
	return n;
}

static void *
rb_read(struct read_block *rb, int sz) {
	if (rb->len < sz) {
		return NULL;
	}

	int ptr = rb->ptr;//获取起始位置
	rb->ptr += sz;//指针移动
	rb->len -= sz;//数据减少
	return rb->buffer + ptr;//返回数据的起始位置
}

```

#### 总结
我们服务间发送消息的时候，先打包，即在底层分配一块内存，等待接收方服务收到消息时，通过解包来获取数据。我们通过解包的过程知道，我们并没有释放发送消息时，打包分配的内存。其实内存的释放，是在处理完这个消息后，被底层释放的。看看下面的代码
```
dispatch_message(struct skynet_context *ctx, struct skynet_message *msg) {
	int type = msg->sz >> MESSAGE_TYPE_SHIFT;//提取信息
	size_t sz = msg->sz & MESSAGE_TYPE_MASK;
	//调用注册的回调函数
	reserve_msg = ctx->cb(ctx, ctx->cb_ud, type, msg->session, msg->source, msg->data, sz);
	if (!reserve_msg) {
		skynet_free(msg->data);//释放内存
	}
```


### 结束