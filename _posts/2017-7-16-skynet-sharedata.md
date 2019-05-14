---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-sharedata              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---

sharedata的缘由是因为开发过程中，无法一次性就把配置信息提前划分好，所以可能多个服务都在共用一个配置表的一部分。如果每个服务都获取这个配置表的所有数据，那么肯定有大量冗余。注意这里说的冗余是数据造成的冗余，跟codecache函数原型的冗余是有区别的。从sharedata的用法来分析。主要流程如下
- 客户端向sharedatad服务申请产生一个共享对象
- 从sharedatad获取对象，并监听这个对象的更新
- 另一个客户端也获取这个对象，并监听更新
- 其中一个客户端向sharedatad发起了更新共享对象
- 其他客户端都自动更新

#### 创建共享对象
看看代码
```
sharedata.new("foobar", { a=1, b= { "hello",  "world" } })
function sharedata.new(name, v, ...)
	skynet.call(service, "lua", "new", name, v, ...)
end

sharedatad.lua

function CMD.new(name, t, ...)
	local dt = type(t)
	local value
	if dt == "table" then
		value = t
	else
		error ("Unknown data type " .. dt)
	end
	newobj(name, value)//产生共享对象
end

local function newobj(name, tbl)
	assert(pool[name] == nil)
	local cobj = sharedata.host.new(tbl)//根据lua表产生一个cobj
	sharedata.host.incref(cobj)
	local v = { value = tbl , obj = cobj, watch = {} }
	objmap[cobj] = v
	pool[name] = v
	pool_count[name] = { n = 0, threshold = 16 }
end
```
sharedata.new创建了一个共享对象。底层实际上是根据lua表达数组部分和哈希部分创建了一个userdata。所谓一个lua表达数组部分和哈希部分是像下面这样。
```
{11,22,33,a=11,b=22}
```
看看底层是如何创建的。我们这里主要是分析通过一个lua表创建一个共享对象。
```
static int
lnewconf(lua_State *L) {
	int ret;
	struct context ctx;
	struct table * tbl = NULL;
	luaL_checktype(L,1,LUA_TTABLE);
	ctx.L = luaL_newstate();
	ctx.tbl = NULL;
	ctx.string_index = 1;	// 1 reserved for dirty flag

	tbl = (struct table *)malloc(sizeof(struct table));//创建一个自定一个的table数据结构
	if (tbl == NULL) {
	
	}
	memset(tbl, 0, sizeof(struct table));
	ctx.tbl = tbl;

	//pconv函数是主要的创建过程
	lua_pushcfunction(ctx.L, pconv);
	lua_pushlightuserdata(ctx.L , &ctx);
	lua_pushlightuserdata(ctx.L , L);

	ret = lua_pcall(ctx.L, 2, 1, 0);

	if (ret != LUA_OK) {
	
	}

	convert_stringmap(&ctx, tbl);

	lua_pushlightuserdata(L, tbl);	//把创建的table数据结构指针压入栈

	return 1;
}

static int
pconv(lua_State *L) {
	struct context *ctx = lua_touserdata(L,1);
	lua_State * pL = lua_touserdata(L, 2);
	int ret;

	lua_settop(L, 0);

	// init L (may throw memory error)
	// create a table for string map
	lua_newtable(L);

	//注意convtable
	lua_pushcfunction(pL, convtable);
	lua_pushvalue(pL,1);
	lua_pushlightuserdata(pL, ctx);

	ret = lua_pcall(pL, 2, 0, 0);
	if (ret != LUA_OK) {

	}

	luaL_checkstack(L, ctx->string_index + 3, NULL);
	lua_settop(L,1);

	return 1;
}
```
下面才是转化的关键函数 convtable。每一个lua表对会转化为一个自定义的一个table数据结构。当然如果表里面又嵌套表，那么也会有多个table数据结构被创建。
```
static int
convtable(lua_State *L) {
	int i;
	struct context *ctx = lua_touserdata(L,2);
	struct table *tbl = ctx->tbl;

	tbl->L = ctx->L;

	int sizearray = lua_rawlen(L, 1);//结算表达数组长度
	if (sizearray) {
		tbl->arraytype = (uint8_t *)malloc(sizearray * sizeof(uint8_t));//分配保存类型的空间
		if (tbl->arraytype == NULL) {
			goto memerror;
		}
		for (i=0;i<sizearray;i++) {
			tbl->arraytype[i] = VALUETYPE_NIL;//初始化类型
		}
		tbl->array = (union value *)malloc(sizearray * sizeof(union value));//分配保存值的空间
		if (tbl->array == NULL) {
			goto memerror;
		}
		tbl->sizearray = sizearray;//保存数组大小
	}
	int sizehash = countsize(L, sizearray);//计算表的哈希长度
	if (sizehash) {
		tbl->hash = (struct node *)malloc(sizehash * sizeof(struct node));//分配节点空间
		if (tbl->hash == NULL) {
			goto memerror;
		}
		for (i=0;i<sizehash;i++) {//初始化
			tbl->hash[i].valuetype = VALUETYPE_NIL;
			tbl->hash[i].nocolliding = 0;
		}
		tbl->sizehash = sizehash;//哈希表长度

		//进行数据填充
		fillnocolliding(L, ctx);
		fillcolliding(L, ctx);
	} else {
		int i;
		for (i=1;i<=sizearray;i++) {
			lua_rawgeti(L, 1, i);
			setarray(ctx, L, -1, i);
			lua_pop(L,1);
		}
	}

	return 0;
```
上面的代码主要是预先分配两部分空间并初始化，主要是为了接下来转化lua表达数组部分和哈希部分。主要这里的类型和值的意思。下面看填充部分
```
static void
fillnocolliding(lua_State *L, struct context *ctx) {
	struct table * tbl = ctx->tbl;
	lua_pushnil(L);
	while (lua_next(L, 1) != 0) {//不断的弹出表中的key-value 也即键值对
		int key;
		int keytype;
		uint32_t keyhash;
		if (!ishashkey(ctx, L, -2, &key, &keyhash, &keytype)) {//数组部分
			setarray(ctx, L, -1, key);
		} else {//哈希部分
			struct node * n = &tbl->hash[keyhash % tbl->sizehash];//根据哈希值分配一个节点
			if (n->valuetype == VALUETYPE_NIL) {
				n->key = key;
				n->keytype = keytype;
				n->keyhash = keyhash;
				n->next = -1;
				n->nocolliding = 1;
				//注意下面这行代码，如果n->valuetype又是table类型，那么需要递归产生共享对象。
				setvalue(ctx, L, -1, n);	// set n->v , n->valuetype
			}
		}
		lua_pop(L,1);
	}
}
```
我们看看设置值得代码 setvalue.如果值得类型不是table，那么直接设置即可。如果是table就会产生递归。
```
static void
setvalue(struct context * ctx, lua_State *L, int index, struct node *n) {
	int vt = lua_type(L, index);
	switch(vt) {
	case LUA_TNIL:
		n->valuetype = VALUETYPE_NIL;
		break;
	case LUA_TNUMBER:
		if (lua_isinteger(L, index)) {
			n->v.d = lua_tointeger(L, index);
			n->valuetype = VALUETYPE_INTEGER;
		} else {
			n->v.n = lua_tonumber(L, index);
			n->valuetype = VALUETYPE_REAL;
		}
		break;
	case LUA_TSTRING: {
		size_t sz = 0;
		const char * str = lua_tolstring(L, index, &sz);
		n->v.string = stringindex(ctx, str, sz);
		n->valuetype = VALUETYPE_STRING;
		break;
	}
	case LUA_TBOOLEAN:
		n->v.boolean = lua_toboolean(L, index);
		n->valuetype = VALUETYPE_BOOLEAN;
		break;
	case LUA_TTABLE: {//值是table类型，
		struct table *tbl = ctx->tbl;//先保存已经产生的对象
		ctx->tbl = (struct table *)malloc(sizeof(struct table));//分配新对象空间
		if (ctx->tbl == NULL) {
			ctx->tbl = tbl;
			luaL_error(L, "memory error");
			// never get here
		}
		memset(ctx->tbl, 0, sizeof(struct table));
		int absidx = lua_absindex(L, index);

		lua_pushcfunction(L, convtable);//又开始调用convtable来创建table数据结构了
		lua_pushvalue(L, absidx);
		lua_pushlightuserdata(L, ctx);

		lua_call(L, 2, 0);

		n->v.tbl = ctx->tbl;
		n->valuetype = VALUETYPE_TABLE;

		ctx->tbl = tbl;

		break;
	}
	default:
		luaL_error(L, "Unsupport value type %s", lua_typename(L, vt));
		break;
	}
}
```
主要代码已经分析完了。大致过程就是根据lua曾传入的lua表中数据的类型，以递归的方式，创建对应的一个自定义的table数据结构。
#### 获取共享对象
获取之前创建的共享对象。
```
local obj = sharedata.query "foobar"

function sharedata.query(name)
	if cache[name] then
		return cache[name]
	end
	local obj = skynet.call(service, "lua", "query", name)//发送获取请求
	local r = sd.box(obj)//设置元表和增加计数引用
	skynet.send(service, "lua", "confirm" , obj)//减少引用
	skynet.fork(monitor,name, r, obj)//开启监听，以便发现共享对象已经发生了改变。
	cache[name] = r
	return r
end

function CMD.query(name)
	local v = assert(pool[name])
	local obj = v.obj
	sharedata.host.incref(obj)//增加引用
	return v.obj
end

function CMD.confirm(cobj)
	if objmap[cobj] then
		sharedata.host.decref(cobj)//减少引用
	end
	return NORET
end

```
发起获取请求后，服务端会增加引用计数，但是服务端如果收到客户端的确认信息"confirm"，又会立即减少引用。我们看看sd.box函数.
```
function conf.box(obj)
	local gcobj = core.box(obj)
	return setmetatable({
		__parent = false,
		__obj = obj,
		__gcobj = gcobj,
		__key = "",
	} , meta)//这里的meta类面包含了_len _pairs _index索引。
end
```
因为这个返回的lua表的元表包含了_len _pairs _index索引。我们这里主要看__index 。因为我们需要获取键值对。
```
function meta:__index(key)
	local obj = getcobj(self)
	local v = index(obj, key)//通过底层访问获取key对应的v
	if type(v) == "userdata" then
		local children = self.__cache
		if children == nil then
			children = {}
			rawset(self, "__cache", children)
		end
		local r = children[key]
		if r then
			return r
		end
		//又产生一个lua表，而且元表是meta。
		r = setmetatable({
			__obj = v,
			__gcobj = self.__gcobj,
			__parent = self,
			__key = key,
		}, meta)
		children[key] = r
		return r
	else
		return v
	end
end
```
这里实际上就是通过obj和key获得v。如果获得的v依然是一个自定义的table，那么又产生一个lua表，并绑定元表，并返回这个表。那么当使用这个返回的表，又可以去取得其他数据了。
##### 开始监听更新
在获取共享对象的时候有这一句
```
skynet.fork(monitor,name, r, obj)//开启监听，以便发现共享对象已经发生了改变。
```
看代码
```
local function monitor(name, obj, cobj)
	local newobj = cobj
	while true do
		newobj = skynet.call(service, "lua", "monitor", name, newobj)//这里会挂起直到收到更新通知
		if newobj == nil then
			break
		end
		sd.update(obj, newobj)//更新
	end
	if cache[name] == obj then
		cache[name] = nil
	end
end
```
发起监听，相当于是注册监听。看代码
```
function CMD.monitor(name, obj)
	local v = assert(pool[name])
	if obj ~= v.obj then
		return v.obj
	end

	local n = pool_count[name].n + 1
	if n > pool_count[name].threshold then
		n = n - check_watch(v.watch)
		pool_count[name].threshold = n * 2
	end
	pool_count[name].n = n

	table.insert(v.watch, skynet.response())//加入监听队列

	return NORET
end
```
##### 某客户端主动发起更新
像下面这样
```
sharedata.update("foobar", { a =2 })

function sharedata.update(name, v, ...)
	skynet.call(service, "lua", "update", name, v, ...)
end

function CMD.update(name, t, ...)
	local v = pool[name]
	local watch, oldcobj
	if v then
		watch = v.watch
		oldcobj = v.obj
		objmap[oldcobj] = true
		sharedata.host.decref(oldcobj)//减少一次引用
		pool[name] = nil
		pool_count[name] = nil
	end
	CMD.new(name, t, ...)//重新生成一个自定义table
	local newobj = pool[name].obj
	if watch then
		sharedata.host.markdirty(oldcobj)//设置脏数据标志
		for _,response in pairs(watch) do//通知监听器，数据更新了
			response(true, newobj)
		end
	end
	collect1min()	-- collect in 1 min//每隔一分钟，垃圾回收一次。
end
```
只有有一个客户端发起更新，那么就会重新生成自定义table。然后通知其他客户端共享对象改变了。其他客户端在绑定监听的时候，就设置了响应更新函数 update。
```
corelib.lua

function conf.update(self, pointer)
	local cobj = self.__obj
	assert(isdirty(cobj), "Only dirty object can be update")
	core.update(self.__gcobj, pointer, { __gcobj = core.box(pointer) })
end

```
### 结束