---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-socketChannel              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
当我们服务器需要主动连接外部主机时，我们需要用到socketChannel。主要有两种跟外界通讯的模式。为了关联请求和响应，一种是通过队列模式，这种方式先发送到请求先收到处理，一种是通过session的模式，不保证顺序，但是保证了一个请求对应一个响应。

#### 模式1 order mode
我们一般发送一个请求方式是这样
```lua
local sc = require "skynet.socketchannel"

local channel = sc.channel {
  host = "127.0.0.1",
  port = 3271,
}

response 这里是一个function
local ret = channel:request(str,response)

```
这里的基本过程是先产生一个channel对象，然后主动建立一条连接，再发送请求，之后通过response函数读取sock接收到的数据，最后把数据返回给request的调用者。看创建channel代码
```
function socket_channel.channel(desc)
	local c = {
		__host = assert(desc.host),
		__port = assert(desc.port),
		__backup = desc.backup,//连接的备用地址
		__auth = desc.auth,//验证函数。如果产生连接后，需要验证，那么要提供验证函数
		__response = desc.response,	-- It's for session mode 这里如果有设置函数 说明书session模式
		__request = {},	-- request seq { response func or session }	-- It's for order mode
		__thread = {}, -- coroutine seq or session->coroutine map
		__result = {}, -- response result { coroutine -> result }
		__result_data = {},
		__connecting = {},
		__sock = false,
		__closed = false,
		__authcoroutine = false,
		__nodelay = desc.nodelay,
	}

	return setmetatable(c, channel_meta)
end
```
我们看发起请求的代码
```
function channel:request(request, response, padding)
	//如果连接没有建立，那么先建立连接。
	assert(block_connect(self, true))	-- connect once
	local fd = self.__sock[1]

	if padding then
		
	else
		if not socket_write(fd , request) then//通过sock发送数据
			sock_err(self)
		end
	end
	//如果是order模式，那么response是函数
	//如果是session模式，那么response是session
	//当然本质上就如名字所示，都是代表响应，只是方式不同
	if response == nil then
		-- no response
		return
	end

	//把这一次请求加入队列，可以认为是登记，挂起等待响应
	return wait_for_response(self, response)
end
```
实际上从发送请求到收到响应的过程如下：
- 主动建立连接
- 开启一个接收协程，通过请求队列里面的请求来决定是否需要阻塞的从sock提取对应的响应数据
- 发送请求
- 把请求登记到请求队列中，等待，直到接收协程获取数据，再把结果返回给request的调用者。

我们先看看登记过程 wait_for_response

```

local function wait_for_response(self, response)
	local co = coroutine.running()
	push_response(self, response, co)//把这次请求加入到请求队列中
	skynet.wait(co)//开始等待接收协程获取数据

	//到这里，已经获得数据了
	local result = self.__result[co]
	self.__result[co] = nil
	local result_data = self.__result_data[co]
	self.__result_data[co] = nil

	if result == socket_error then
		if result_data then
			error(result_data)
		else
			error(socket_error)
		end
	else
		assert(result, result_data)
		return result_data//返回数据给requst的调用者
	end
end

local function push_response(self, response, co)
	if self.__response then//session mode
		-- response is session
		self.__thread[response] = co
	else//order mode
		-- response is a function, push it to __request
		//插入队列
		table.insert(self.__request, response)
		table.insert(self.__thread, co)
		if self.__wait_response then //如果接收协程阻塞了，也就是发现请求队列中没有请求了。
			skynet.wakeup(self.__wait_response)//这时候，因为有新的请求加入，所以可以唤醒接收协程。
			self.__wait_response = nil
		end
	end
end

```

我们再看看建立连接的过程
```
local function block_connect(self, once)
	local r = check_connection(self)//检查连接是否已经建立
	if r ~= nil then
		return r
	end
	local err

	if #self.__connecting > 0 then
		-- connecting in other coroutine
		local co = coroutine.running()
		table.insert(self.__connecting, co)
		skynet.wait(co)
	else
		self.__connecting[1] = true
		err = try_connect(self, once)//没有建立那么 开始建立
		self.__connecting[1] = nil
		for i=2, #self.__connecting do
			local co = self.__connecting[i]
			self.__connecting[i] = nil
			skynet.wakeup(co)
		end
	end

	r = check_connection(self)
	if r == nil then
		skynet.error(string.format("Connect to %s:%d failed (%s)", self.__host, self.__port, err))
		error(socket_error)
	else
		return r
	end
end

local function try_connect(self , once)
	local t = 0
	while not self.__closed do
		local ok, err = connect_once(self)//这里建立连接的主要函数
		if ok then
			if not once then
				skynet.error("socket: connect to", self.__host, self.__port)
			end
			return
		elseif once then
			return err
		else
			skynet.error("socket: connect", err)
		end
		if t > 1000 then
			skynet.error("socket: try to reconnect", self.__host, self.__port)
			skynet.sleep(t)
			t = 0
		else
			skynet.sleep(t) //如果不是只连接一次 那么每个一秒重连一次
		end
		t = t + 100
	end
end

local function connect_once(self)
	if self.__closed then
		return false
	end
	assert(not self.__sock and not self.__authcoroutine)
	local fd,err = socket.open(self.__host, self.__port)//发起主动连接
	if not fd then
		fd = connect_backup(self)//连接不成功 启动备用地址继续连接
		if not fd then
			return false, err
		end
	end
	if self.__nodelay then//关闭tcp的nagle算法
		socketdriver.nodelay(fd)
	end

	self.__sock = setmetatable( {fd} , channel_socket_meta )
	//开启一个接收协程，用来获取sock的数据
	self.__dispatch_thread = skynet.fork(dispatch_function(self), self)

	//连接成功后，开始验证
	if self.__auth then
		self.__authcoroutine = coroutine.running()
		local ok , message = pcall(self.__auth, self)
		if not ok then
			close_channel_socket(self)
			if message ~= socket_error then
				self.__authcoroutine = false
				skynet.error("socket: auth failed", message)
			end
		end
		self.__authcoroutine = false
		if ok and not self.__sock then
			-- auth may change host, so connect again
			return connect_once(self)
		end
		return ok
	end

	return true
end
```
接收协程对应两种模式
```
local function dispatch_function(self)
	if self.__response then
		return dispatch_by_session
	else
		return dispatch_by_order
	end
end
```
我们这里暂时只看 order模式
```
local function dispatch_by_order(self)
	while self.__sock do
		local func, co = pop_response(self)//弹出请求的登记
		//待用func函数提取sock收到的数据
		local ok, result_ok, result_data, padding = pcall(func, self.__sock)

	if ok then
		self.__result[co] = result_ok
		if result_ok and self.__result_data[co] then
			table.insert(self.__result_data[co], result_data)
		else
			//保存获取的数据
			self.__result_data[co] = result_data
		end
		//唤醒之前等待在这个请求上的协程
		skynet.wakeup(co)
...


local function pop_response(self)
	while true do
		//取出一个请求
		local func,co = table.remove(self.__request, 1), table.remove(self.__thread, 1)
		if func then
			return func, co
		end
		//如果没有请求，则等待新请求加入队列
		self.__wait_response = coroutine.running()
		skynet.wait(self.__wait_response)
	end
end
}
```
#### 模式2 session mode
这个跟order mode唯一不同的地方在于，响应是通过session来匹配的。我们建立channel 的时候，配置里面要设置 `__response `成员。因为接收协程就是通过这个字段来分区模式的。具体的代码都是类似，这里不再讨论。

#### 总结
发出请求数据，同时把请求登记到请求队列，这样接收协程通过登记，可以找到请求对应的读取sock的函数，组后返回数据给request的调用者。
### 结束