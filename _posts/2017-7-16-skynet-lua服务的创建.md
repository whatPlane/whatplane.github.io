---
layout:     post                    # 使用的布局（不需要改）
title:      skynet_lua服务的创建              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
我们创建一个lua服务一般是这样 `skynet.newservice("debug_console",8000)`。看看里面的代码

```
function skynet.newservice(name, ...)
	return skynet.call(".launcher", "lua" , "LAUNCH", "snlua", name, ...)
end
```

这说明创建lua服务是直接把消息发送给 `.launcher`服务，也就是 `.launcher`来完成创建的。而.launcher服务是在bootstrap里面完成创建的。看看 bootstrap.lua的初始化代码
```
	local launcher = assert(skynet.launch("snlua","launcher"))
	skynet.name(".launcher", launcher)
```
再看skynet.launch代码的实现
```
function skynet.launch(...)
	local addr = c.command("LAUNCH", table.concat({...}," "))
	if addr then
		return tonumber("0x" .. string.sub(addr , 2))
	end
end
```
看看c.command的底层代码。发现这里就是在创建了 launcher 服务。
```
static const char *
cmd_launch(struct skynet_context * context, const char * param) {
	size_t sz = strlen(param);
	char tmp[sz+1];
	strcpy(tmp,param);
	char * args = tmp;
	char * mod = strsep(&args, " \t\r\n");
	args = strsep(&args, "\r\n");
	struct skynet_context * inst = skynet_context_new(mod,args);//创建launcher服务
	if (inst == NULL) {
		return NULL;
	} else {
		id_to_hex(context->result, inst->handle);
		return context->result;
	}
}

```
再看看launcher服务是怎么 提供创建其他服务的。
```
function command.LAUNCH(_, service, ...) //处理创建新服务的请求
	launch_service(service, ...)
	return NORET
end

local function launch_service(service, ...)

	local inst = skynet.launch(service, param)//产生新服务
	
end
```

### 结束