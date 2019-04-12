---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-lua层网络监听过程              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
之前我们讨论了 [epoll网络监听](https://whatplane.github.io/2017/07/17/skynet-epoll%E5%8F%91%E8%B5%B7%E7%9B%91%E5%90%AC/) 这一次我们讨论lua层监听的相关细节。我们根据下面这段代码分析
```
local skynet    = require "skynet"
local socket    = require "skynet.socket"

skynet.start(function()
    local addr = "0.0.0.0:8001"
    skynet.error("listen " .. addr)
    local lID = socket.listen(addr)
    assert(lID)
    socket.start(lID, accept)
end)
```

#### socket.start发送命令给底层
我们服务的初始化函数start在执行到 socket.start(lID, accept)时，这个协程被挂起。
```
function socket.start(id, func)
	driver.start(id)
	return connect(id, func)
end

local function connect(id, func)
	...	
	suspend(s)//让出执行权
	if s.connected then
		return id
	else
		socket_pool[id] = nil
		return nil, err
	end

```
suspend挂起的原因是"SLEEP"。也就是挂起等待底层告诉lua层，监听是否设置完成。
```
local function suspend(s)
	assert(not s.co)
	s.co = coroutine.running()
	skynet.wait(s.co)//进入睡眠状态
	
end

function skynet.wait(co)
	local session = c.genid()
	local ret, msg = coroutine_yield("SLEEP", session)//让出执行权
	co = co or coroutine.running()
	sleep_session[co] = nil
	session_id_coroutine[session] = nil
end

```
这个时候，start函数所在的协程挂起了，skynet.suspend根据挂起的原因"SLEEP"，保存这个休眠协程。
```
function suspend(co, result, command, param, size)
	elseif command == "SLEEP" then
		session_id_coroutine[param] = co
		sleep_session[co] = param//保存起来
```
#### 底层响应 socket.start
当底层把监听sock加入到epoll后，会push一个本地消息到监听服务所在地队列。当然这个本地消息的类型是网络类型，即PTYPE_SOCKET。而这个已经被转化为本地消息的网络消息，对应的网络消息内部是SKYNET_SOCKET_TYPE_CONNECT。收到消息后会在raw_dispatch_message里面启动一个新的协程 x，这跟之前start函数对应的协程是不同的。
```
local function raw_dispatch_message(prototype, msg, sz, session, source)

	if prototype == 1 then
	else//从这里执行
		local p = proto[prototype]
		local f = p.dispatch
		if f then
			local ref = watching_service[source]
			if ref then
				watching_service[source] = ref + 1
			else
				watching_service[source] = 1
			end
			local co = co_create(f)//产生一个新协程
			session_coroutine_id[co] = session
			session_coroutine_address[co] = source
			suspend(co, coroutine_resume(co, session,source, p.unpack(msg,sz)))



```
下面解释产生一个新协程的原因。再次看下面的 co_create代码。
```
local function co_create(f)
	local co = table.remove(coroutine_pool)
	if co == nil then
		co = coroutine.create(function(...)
			f(...)//start函数让出的位置在这个里面，所以coroutine_pool并没有回收这个协程。
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
新协程根据注册的dispatch函数，会唤醒之前睡眠的协程
```
skynet.register_protocol {
	name = "socket",
	id = skynet.PTYPE_SOCKET,	-- PTYPE_SOCKET = 6
	unpack = driver.unpack,
	dispatch = function (_, _, t, ...)
		socket_message[t](...)
	end
}


-- SKYNET_SOCKET_TYPE_CONNECT = 2
socket_message[2] = function(id, _ , addr)
	local s = socket_pool[id]
	if s == nil then
		return
	end
	-- log remote addr
	s.connected = true
	wakeup(s)//唤醒之前睡眠的协程
end

```
唤醒实际上是把之前start函数对应的协程加入到 wakeup_queue **唤醒队列** ，我们看代码
```
local function wakeup(s)
	local co = s.co
	if co then
		s.co = nil
		skynet.wakeup(co)
	end
end

function skynet.wakeup(co)
	if sleep_session[co] then
		table.insert(wakeup_queue, co)//加入到唤醒队列，之后即将被唤醒
		return true
	end
end
```
之后 x 协程执行让出。让出参数是"EXIT"。suspend之后唤醒 wakeup_queue。这样之前的start协程可以继续执行了
```
function suspend(co, result, command, param, size)
	...
	elseif command == "EXIT" then
	...

	dispatch_wakeup()

local function dispatch_wakeup()
	local co = table.remove(wakeup_queue,1)
	if co then
		local session = sleep_session[co]
		if session then
			session_id_coroutine[session] = "BREAK"
			return suspend(co, coroutine_resume(co, false, "BREAK"))//继续执行start对应的协程
		end
	end
end
```
继续执行之前协程的位置
```
function skynet.wait(co)
	local session = c.genid()
	local ret, msg = coroutine_yield("SLEEP", session)//是时候返回了
	co = co or coroutine.running()
	sleep_session[co] = nil
	session_id_coroutine[session] = nil
end
```
因为 skynet.wait是 `local function connect(id, func)`函数调用的，所以回到本文最开始的suspend地方，接着执行，最后我们 的 sock.start返回一个id

#### 图示
这里的图不一定准确，但主要是有助于理解。红色是协程让出的时刻。绿色代表开始执行时刻

![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190412161602-min.png)

### 结束