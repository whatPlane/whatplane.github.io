---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-redis              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
skynet的跟redis的通讯是通过sockchannel实现的。而且是工作在 order mode。我们直接看看redis在skyent里面的基本使用。主要分为三部分
- 基本请求命令
- 订阅和发布
- 管道功能

下面这段代码反映了redis的基本使用
```
local skynet = require "skynet"
local redis = require "skynet.db.redis"

local conf = {
	host = "127.0.0.1" ,
	port = 6379 ,
	db = 0
}

local function watching()
	local w = redis.watch(conf)//建立一条连接，用于接收订阅的频道
	w:subscribe "foo"
	w:psubscribe "hello.*"
	while true do
		print("Watch", w:message())
	end
end

skynet.start(function()
	skynet.fork(watching)//创建一个订阅协程
	local db = redis.connect(conf)//创建一个连接用于发送请求命令

	db:set("A", "hello")//设置命令
	
	for i=1,10 do
		db:publish("foo", i)//发布消息
	end
	...

end)
```
#### 基本请求命令

从这段代码开始`local db = redis.connect(conf)`.代码在redis.lua中
```
function redis.connect(db_conf)
	local channel = socketchannel.channel {
		host = db_conf.host,
		port = db_conf.port or 6379,
		auth = redis_login(db_conf.auth, db_conf.db),//连接建立后，立即验证
		nodelay = true,
	}
	-- try connect first only once
	channel:connect(true)//建立连接，并创建接收协程
	return setmetatable( { channel }, meta )
end

local function redis_login(auth, db)
	if auth == nil and db == nil then
		return
	end
	return function(so)
		if auth then //密码验证
			so:request(compose_message("AUTH", auth), read_response)
		end
		if db then//数据库选择
			so:request(compose_message("SELECT", db), read_response)
		end
	end
end
```
在连接成功后，会发送验证请求，收到服务器响应后，调用read_response 来解析响应结果。实际上我们在执行`db:set("A", "hello")`等类似的命令时，也是先发送命令然后解析服务器的响应。下面这行代码就是我们执行命令时的主要代码
```
setmetatable(command, { __index = function(t,k)
	local cmd = string.upper(k)
	local f = function (self, v, ...)
		if type(v) == "table" then 
			return self[1]:request(compose_message(cmd, v), read_response)
		else
			return self[1]:request(compose_message(cmd, {v, ...}), read_response)
		end
	end
	t[k] = f
	return f
end})

```
我们看到跟刚才的讨论的验证命令时一致的。这里我们主要看看compose_message和read_response函数。讨论前先我们需要了解到跟redis通讯是需要满足一定的协议规则的。比如我们发送`SET key value`命令，实际上代码中会转化为下面的请求数据。把这个请求认为是三个参数。

```
    *3 表示总共有三个参数
    $3 这里的'SET'长度是3个字节
    SET
    $3 # 这里 key 长度是3个字节
    key
    $5 # 这里 value 长度是5个字节
    value

   
```
redis把SET也当作是一个参数。所以我们是有三个参数。每个 数据单元 用 `\n\r` 分隔开.这个命令的实际传输数据是这样：`*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n`

所以我们发送命令前，需要先把参数进行组合。也就是compose_message函数做的事情。我们看代码
```
local function make_cache(f)
	return setmetatable({}, {
		__mode = "kv",
		__index = f,
	})
end

local header_cache = make_cache(function(t,k)
		local s = "\r\n$" .. k .. "\r\n" //参数长度
		t[k] = s
		return s
	end)

local command_cache = make_cache(function(t,cmd)
		local s = "\r\n$"..#cmd.."\r\n"..cmd:upper()//命令长度 + 命令
		t[cmd] = s
		return s
	end)

local count_cache = make_cache(function(t,k)
		local s = "*" .. k //参数个数
		t[k] = s
		return s
	end)

local function compose_message(cmd, msg)
	local t = type(msg)
	local lines = {}

	if t == "table" then 
		lines[1] = count_cache[#msg+1]
		lines[2] = command_cache[cmd]
		local idx = 3
		for _,v in ipairs(msg) do
			v= tostring(v)
			lines[idx] = header_cache[#v]
			lines[idx+1] = v
			idx = idx + 2
		end
		lines[idx] = "\r\n"
	else
		msg = tostring(msg)
		lines[1] = "*2"
		lines[2] = command_cache[cmd]
		lines[3] = header_cache[#msg]
		lines[4] = msg
		lines[5] = "\r\n"
	end

	return lines
end
```

我们看看read_response函数
```
local function read_response(fd)
	local result = fd:readline "\r\n"
	local firstchar = string.byte(result)
	local data = string.sub(result,2)
	return redcmd[firstchar](fd,data)
end

redcmd[43] = function(fd, data) -- '+'//状态回复
	return true,data
end

redcmd[45] = function(fd, data) -- '-' //错误回复
	return false,data //返回false表示有错误
end

redcmd[36] = function(fd, data) -- '$' //批量回复
	local bytes = tonumber(data)//获取数据大小
	if bytes < 0 then
		return true,nil
	end
	local firstline = fd:read(bytes+2)//读取后续数据
	return true,string.sub(firstline,1,-3)//丢弃最后的两个字节
end

```
实际上redis服务器返回的数据是以`\r\n`分割的。一个请求的响应数据的第一个字符代表着不同的意义。
- 状态回复（status reply）的第一个字节是 "+"
- 错误回复（error reply）的第一个字节是 "-"
- 整数回复（integer reply）的第一个字节是 ":"
- 批量回复（bulk reply）的第一个字节是 "$"
- 多条批量回复（multi bulk reply）的第一个字节是 "*"

最后我们requset调用者获得的就是   redcmd 返回的第二个参数的数据

#### 发布订阅
我们看看订阅代码
```
local function watching()
	local w = redis.watch(conf)//建立一条连接，用于接收订阅的频道
	w:subscribe "foo"
	w:psubscribe "hello.*"
	while true do
		print("Watch", w:message())
	end
end
```
上面的代码主要建立连接，然后订阅了连个频道，最后不断的接收频道发布的消息。首先看初始化过程
```

function redis.watch(db_conf)
	local obj = {
		__subscribe = {},
		__psubscribe = {},
	}
	local channel = socketchannel.channel {
		host = db_conf.host,
		port = db_conf.port or 6379,
		auth = watch_login(obj, db_conf.auth),
		nodelay = true,
	}
	obj.__sock = channel

	-- try connect first only once
	channel:connect(true)
	return setmetatable( obj, watchmeta )
end
```
再看订阅代码
```
local function watch_func( name )
	local NAME = string.upper(name)
	watch[name] = function(self, ...)
		local so = self.__sock
		for i = 1, select("#", ...) do
			local v = select(i, ...)
			so:request(compose_message(NAME, v))//发送请求
		end
	end
end

watch_func "subscribe"
watch_func "psubscribe"
watch_func "unsubscribe"
watch_func "punsubscribe"
```
实际上发送订阅，也是发送类似 `subscribe foo`这种字符串给服务器。但是这里发送请求时，没有提供响应函数read_response。因为订阅一个频道，后面可以接收这个频道的所有推送，不是一个请求对应一个响应的模式。所以才会有下面的代码
```
while true do
		print("Watch", w:message())
	end
```
不断的取出推送消息。
```
function watch:message()
	local so = self.__sock
	while true do
		local ret = so:response(read_response)//这里表示登记一个请求到队列，表示我们希望阻塞读取推送
		local type , channel, data , data2 = ret[1], ret[2], ret[3], ret[4]
		if type == "message" then//收到发布消息
			return data, channel
		elseif type == "pmessage" then
			return data2, data, channel
		elseif type == "subscribe" then//订阅成功
			self.__subscribe[channel] = true
		elseif type == "psubscribe" then
			self.__psubscribe[channel] = true
		elseif type == "unsubscribe" then//取消订阅成功
			self.__subscribe[channel] = nil
		elseif type == "punsubscribe" then
			self.__psubscribe[channel] = nil
		end
	end
end
```
这里so:response(read_response)实际上是添加请求到队列，当然不是指发送一个请求给服务器。可以理解为添加一个回调函数到队列，目的是唤醒接收协程，去查看是否有推送消息。看代码sockChannel.lua
```
function channel:response(response)
	assert(block_connect(self))

	return wait_for_response(self, response)
end

local function wait_for_response(self, response)
	local co = coroutine.running()
	push_response(self, response, co)//push动作
	skynet.wait(co)//挂起自己

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
		return result_data
	end
end

```
再看发布消息代码
```
db:publish("foo", i)//发布消息
```
实际上这个代码跟 `db:set("A", "hello")`的过程没有区别。都是发送一个请求给服务器
#### 管道
redis客户端和服务器通信是这样的: 客户端发起一个请求命令，然后阻塞，等待服务处理后，阻塞返回，获取响应数据。redis管道可以一次性发送多个请求命令，然后等待服务器执行返回结果。注意区别类似mget这样的命令。我们看下面的代码
```
mset java good lua ok php nice //(mset key1 value1 key2 value2 ...)
``` 
mset本身已经是批量处理了，而管道可以一次性发送多个 mset这样的命令。流水线的作用主要是把多个命令一次性发送，减少多个命令单独发送时的网络耗时。管道的使用
```
print("test table")
		local ret = db:pipeline({
			{"hincrby", "hello", 1, 1},
			{"del", "hello"},
			{"hmset", "hello", 1, 1, 2, 2, 3, 3},
			{"hgetall", "hello"},
		}, {})	-- offer a {} for result

		print(ret[1].out)
		print(ret[2].out)
		print(ret[3].out)

		for k, v in pairs(read_table(ret[4].out)) do
			print(k, v)
		end
```
看看代码
```
function command:pipeline(ops,resp)
	assert(ops and #ops > 0, "pipeline is null")

	local fd = self[1]

	local cmds = {}//收集所有操作
	for _, cmd in ipairs(ops) do
		compose_table(cmds, cmd)
	end

	if resp then
		return fd:request(cmds, function (fd)
			for i=1, #ops do//发送几个请求命令，就读取响应几次
				local ok, out = read_response(fd)
				table.insert(resp, {ok = ok, out = out})
			end
			return true, resp
		end)
	else
		return fd:request(cmds, function (fd)
			local ok, out
			for i=1, #ops do
				ok, out = read_response(fd)
			end
			-- return last response
			return ok,out
		end)
	end
end
```

### 结束