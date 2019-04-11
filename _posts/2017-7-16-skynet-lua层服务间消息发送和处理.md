---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-lua层服务间消息发送和处理              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---

我这里主要是通过bootstrap服务，launch服务，来说明lua服务间是怎么发送和接收消息的。在skynet节点启动的时候，我们首先会启动bootstrap服务，而在bootstrap服务的初始化函数里面会启动launch服务，之后又通过launch服务启动其他服务（比如启动slave服务）。关于处理消息的协程的执行流转换，需要一直关注协程的创建代码。如下
```
local function co_create(f)
	local co = table.remove(coroutine_pool)
	if co == nil then
		co = coroutine.create(function(...)
			f(...)
			while true do
				f = nil
				coroutine_pool[#coroutine_pool+1] = co
				f = coroutine_yield "EXIT" 
				f(coroutine_yield())
			end
		end)
	else
		coroutine_resume(co, f)
	end
	return co
end
```


#### bootstrap服务
bootstrap服务的执行start函数的协程在调用 skynet.call请求launch服务创建新服务时，即`pcall(skynet.newservice, "cdummy")`时，当前协程被挂起。skynet.call代码如下

```
function skynet.call(addr, typename, ...)
	local p = proto[typename]
	local session = c.send(addr, p.id , nil , p.pack(...))//发送请求给launch服务
	if session == nil then
		error("call to invalid address " .. skynet.address(addr))
	end
	return p.unpack(yield_call(addr, session))//让出当前协程执行权
end

```
协程挂起的位置在start里面。如下图。skynet.dispatch_message函数继续后续流程
![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190410172405.png)
#### launch服务
launch服务启动服务后，收到bootstrap服务发送过来的type为lua类型消息，最终会调用command.LAUNCH处理。此时处理该消息的协程，在创建一个新服务后，会让出本协程的执行权。
```
function skynet.response(pack)
	pack = pack or skynet.pack
	return coroutine_yield("RESPONSE", pack)//让出执行权
end


local function launch_service(service, ...)
	local param = table.concat({...}, " ")
	local inst = skynet.launch(service, param)
	local response = skynet.response()//让出该线程的执行权
	if inst then
		services[inst] = service .. " " .. param
		instance[inst] = response
	else
		response(false)
		return
	end
	return inst
end


function command.LAUNCH(_, service, ...)
	launch_service(service, ...)
	return NORET
end
```
我们知道这个处理收到请求消息的协程的是被raw_dispatch_message函数内的suspend函数唤醒的，此时让出执行权，那么suspend函数根据让出时的返回值"RESPONSE"，决定下一步继续执行之前的协程。代码如下
```
function suspend(co, result, command, param, size)
elseif command == "RESPONSE" then
		local co_session = session_coroutine_id[co]
		local co_address = session_coroutine_address[co]
		if session_response[co] then
			error(debug.traceback(co))
		end
		local f = param//f=skyent.pack 打包
		local function response(ok, ...)
				xxxxxx
		end
		watching_service[co_address] = watching_service[co_address] + 1
		session_response[co] = true
		unresponse[response] = true

		//co协程刚刚让出执行权，现在又被唤醒了
		return suspend(co, coroutine_resume(co, response))//主要给协程传递了一个参数response
```
所以该协程从 local function launch_service(service, ...)让出的位置，继续执行。最后返回一个 "EXIT" ，协程让出后，skynet.dispatch_message继续后续流程。

#### cdummy服务
通过launch新创建的cdummy服务，完成start处理后，会发送一个消息给launch服务。就是如下代码所示
```
function skynet.init_service(start)
	local ok, err = skynet.pcall(start)
	if not ok then
		skynet.error("init service failed: " .. tostring(err))
		skynet.send(".launcher","lua", "ERROR")
		skynet.exit()
	else
		skynet.send(".launcher","lua", "LAUNCHOK")//这个就是调用完start函数，发送给launch服务的
	end
end
```
launch服务收到这个消息后，就会回应之前bootstrap服务发出的创建新服务的请求。注意response 发送的消息类型是 skynet.PTYPE_RESPONSE。

```
function command.LAUNCHOK(address)
	-- init notice
	local response = instance[address]
	if response then
		response(true, address)//这里会发送消息给bootstrap服务。
		instance[address] = nil
	end

	return NORET
end

//response这个函数主要做的事情是

c.send(co_address, skynet.PTYPE_RESPONSE, co_session, f(...))

```
bootstrap服务此时，收到launch服务发送的响应消息。通过seesion获得挂起在start里面的协程，通过suspend唤醒这个协程继续执行。
#### 总结
上面主要分析了 skynet.call的调用方和接收方的处理过程。这是大概的流程演示图
![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190411174234-min.png)
### 结束