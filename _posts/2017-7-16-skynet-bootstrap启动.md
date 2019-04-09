---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-bootstrap              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
我们看example目录下config ，注意 `bootstrap = "snlua bootstrap"` 表示我们即将即启动一个snlua服务，这个服务的消息处理脚本是 bootstrap文件
```
include "config.path"

start = "main"	-- main script
bootstrap = "snlua bootstrap"	-- The service for bootstrap

```
我们在skynet_start.c 的开始函数里面调用bootstrap,来启动一个lua服务。
```
skynet_start(struct skynet_config * config) {
	bootstrap(ctx, config->bootstrap);
}

static void
bootstrap(struct skynet_context * logger, const char * cmdline) {
	int sz = strlen(cmdline);
	char name[sz+1];
	char args[sz+1];
	sscanf(cmdline, "%s %s", name, args);
	struct skynet_context *ctx = skynet_context_new(name, args);

}


```
一个服务的启动的基本过程如下所示。注意param 表示我们具体的服务对应的文件
```
struct skynet_context * 
skynet_context_new(const char * name, const char *param) {
	int r = skynet_module_instance_init(mod, inst, ctx, param);
｝

skynet_module_instance_init(struct skynet_module *m, void * inst, struct skynet_context *ctx, const char * parm) {
	return m->init(inst, ctx, parm);
}


```
因为我们启动但是一个 snlua服务，其对应的模块文件在 service_snlua.c。我们看init函数。
```
snlua_init(struct snlua *l, struct skynet_context *ctx, const char * args) {
	int sz = strlen(args);
	char * tmp = skynet_malloc(sz);
	memcpy(tmp, args, sz);
	skynet_callback(ctx, l , launch_cb);
	const char * self = skynet_command(ctx, "REG", NULL);
	uint32_t handle_id = strtoul(self+1, NULL, 16);
	// it must be first message
	skynet_send(ctx, 0, handle_id, PTYPE_TAG_DONTCOPY,0, tmp, sz);
	return 0;
}
```
这里先绑定了服务对应的回调函数，然后注册了这个服务，最后发送了一个消息给自己。因为回调函数是 launch_cb，我们看接下来收到第一个消息的处理
```
static int
launch_cb(struct skynet_context * context, void *ud, int type, int session, uint32_t source , const void * msg, size_t sz) {
	assert(type == 0 && session == 0);
	struct snlua *l = ud;
	skynet_callback(context, NULL, NULL);//解除回调函数绑定
	int err = init_cb(l, context, msg, sz);//开始初始化
	if (err) {
		skynet_command(context, "EXIT", NULL);
	}

	return 0;
}
```
首先解除了之前回调函数和服务的绑定。然后调用真正的初始化函数 init_cb 
```
static int
init_cb(struct snlua *l, struct skynet_context *ctx, const char * args, size_t sz) {

	const char * loader = optstring(ctx, "lualoader", "./lualib/loader.lua");

	int r = luaL_loadfile(L,loader);

	lua_pushlstring(L, args, sz);
	r = lua_pcall(L,1,0,1);//加载 bootstrap.lua
```
上面的函数做了很多初始化的工作，比如环境设置，查找路径设置等。之后就是加载 loader.lua 文件 ，然后加载 我们服务对应的文件 bootstrap.lua。我们看看 bootstrap的代码
```
local skynet = require "skynet"
local harbor = require "skynet.harbor"
require "skynet.manager"	-- import skynet.launch, ...
local memory = require "skynet.memory"

skynet.start(
	
	function()
		......
	end
)
```
也就是加载 bootstrap 主要是执行了 skynet.start 函数。skynet.start 这个函数主要是提供注册功能。我们看看skynet.start
```
function skynet.start(start_func)
	c.callback(skynet.dispatch_message)//告诉底层，收到消息后应该调用哪个lua函数
	skynet.timeout(0, function()
		skynet.init_service(start_func)
	end)
end

function skynet.timeout(ti, func)
	local session = c.intcommand("TIMEOUT",ti) //注册定时器
	assert(session)
	local co = co_create(func)//预先分配一个协程
	assert(session_id_coroutine[session] == nil)
	session_id_coroutine[session] = co //保存起来
end

```
这要是绑定了服务对应的回调函数，通过启动定时器，进行初始化。具体看看 c.callback
```
static int
lcallback(lua_State *L) {
	
	if (forward) {
		skynet_callback(context, gL, forward_cb);
	} else {
		skynet_callback(context, gL, _cb);//注册回调函数
	}

	return 0;
}


_cb(struct skynet_context * context, void * ud, int type, int session, uint32_t source, const void * msg, size_t sz) {

	lua_State *L = ud;
	int trace = 1;
	int r;
	int top = lua_gettop(L);
	if (top == 0) {
		lua_pushcfunction(L, traceback);
		lua_rawgetp(L, LUA_REGISTRYINDEX, _cb);
	} else {
		assert(top == 2);
	}
	lua_pushvalue(L,2);

	lua_pushinteger(L, type);
	lua_pushlightuserdata(L, (void *)msg);
	lua_pushinteger(L,sz);
	lua_pushinteger(L,  );
	lua_pushinteger(L, source);

	r = lua_pcall(L, 5, 0 , trace);//最终调用我们的lua层函数 skynet.dispatch_message
｝
```
最后bootstrap服务因为定时器回调，开始处理调用skynet.dispatch函数。至于skynet.dispatch是怎么分发收到的消息，请看 [lua层的消息分发](https://whatplane.github.io/2017/07/17/skynet-lua%E5%B1%82%E6%B6%88%E6%81%AF%E5%A4%84%E7%90%86/) 。这个函数最终调用我们的初始化时注册的函数start_func。 现在我们回到 bootstrap.lua 看看初始化到底做了什么。

### 结束