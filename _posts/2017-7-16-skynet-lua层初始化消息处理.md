---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-lua层初始化消息处理              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
#### 前置知识 协程 
我们这里就把协程叫做线程吧。主要有三个函数
- coroutine.create(fun) 调用这个函数会产生一个 线程对象，但是里面fun不会立即执行
- coroutine.resume(co,...)  这里会唤醒co线程，同时把参数传递给线程，代码执行流指向co线程的的第一行代码。此时coroutine.resume 处于阻塞状态。只有当co线程返回，coroutine.resume函数才会返回。co线程返回分为正常返回和异常返回。
  - 正常返回 co线程执行完最后一行代码 或者 co线程调用 coroutine.yied（...） 主动让出。此时coroutine.resume的返回值是 true + ...
  - 异常返回 co线程执行出错。此时coroutine.resume返回值是 false + errcode desprition 
- coroutine.yied(...) 表示主动让出本线程的执行权限，直到外部调用 coroutine.resume 唤醒本线程，才会继续从让出点往下执行。被唤醒时，coroutine.yied的返回值是 coroutine.resume的传入的参数（当然不包括线程对象本身）。

下面这段代码可以帮助理解。

```
function foo (a)
   print("foo", a)
   return coroutine.yield(2*a)
 end

 co = coroutine.create(function (a,b)
       print("co-body", a, b)
       local r = foo(a+1)
       print("co-body", r)
       local r, s = coroutine.yield(a+b, a-b)
       print("co-body", r, s)
       return b, "end"
 end)

 print("main", coroutine.resume(co, 1, 10))
 print("main", coroutine.resume(co, "r"))
 print("main", coroutine.resume(co, "x", "y"))
 print("main", coroutine.resume(co, "x", "y"))

执行后的结果
co-body	1	10
foo	2
main	true	4
co-body	r
main	true	11	-9
co-body	x	y
main	true	10	end
main	false	cannot resume dead coroutine


```
skynet内部稍微封装了协程，提供了自己的几个协成专用函数。

#### lua层接受消息后的处理过程
我们的lua层的服务创建后，lua服务的初始化函数是通过skynet.start注册的。我们看这个代码

```
function skynet.init_service(start)
	local ok, err = skynet.pcall(start)
	if not ok then
		skynet.error("init service failed: " .. tostring(err))
		skynet.send(".launcher","lua", "ERROR")
		skynet.exit()
	else
		skynet.send(".launcher","lua", "LAUNCHOK")
	end
end

function skynet.start(start_func)
	c.callback(skynet.dispatch_message)
	skynet.timeout(0, function()
		skynet.init_service(start_func)
	end)
end
```
这里定时器函数skynet.timeout 会产生一个协程，这个协程对应的函数是 skynet.init_service。也就是 skynet.dispatch_message收到定时器消息后，首先就要唤醒这个协程。我们以bootstrap.lua代表的服务为例，讨论收到的一个lua定时器消息。
![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190410172405.png)


```
local function co_create(f)
	local co = table.remove(coroutine_pool)
	if co == nil then
		co = coroutine.create(function(...)
			f(...)//这里就是我们的skynet.init_service函数
			while true do
				f = nil
				coroutine_pool[#coroutine_pool+1] = co
				f = coroutine_yield "EXIT" //这里是第一次挂起后的返回值 exit
				f(coroutine_yield())
			end
		end)
	else
		coroutine_resume(co, f)
	end
	return co
end
```

唤醒协程是在 suspend(co,coroutine_resume(co))
```
function skynet.dispatch_message(...)
	local succ, err = pcall(raw_dispatch_message,...)//最先处理收到的消息对应的协程
	while true do//遍历队列
		local key,co = next(fork_queue)
		fork_queue[key] = nil
		local fork_succ, fork_err = pcall(suspend,co,coroutine_resume(co))//唤醒协程

	end
	assert(succ, tostring(err))
end

local function raw_dispatch_message(prototype, msg, sz, session, source)
	-- skynet.PTYPE_RESPONSE = 1, read skynet.h
	if prototype == 1 then //定时器消息的type是 1
		local co = session_id_coroutine[session]
	
			session_id_coroutine[session] = nil
			suspend(co, coroutine_resume(co, true, msg, sz))//唤醒协程，同时此处执行流挂起
		end
```
在bootstrap服务的第一个定时器消息处理中，最终调用了我们通过skynet.start注册初始化函数,上图中我用红色标志出来了。其他部分因为条件为满足而没有执行。比如 下面的函数调用
```
dispatch_wakeup()
dispatch_error_queue()

```

#### lua服务间的消息发送和接收处理
这个可以从bootstrap服务中启动的launch服务来讨论。
### 结束