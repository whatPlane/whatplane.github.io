---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-http-2              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---

之前讨论了http的基本原理。现在看skynet中的web服务。基本流程是跟我们产生一个监听服务类似。主要麻烦的地方在于解析收到的http请求的数据。这里需要特别注意 http消息头中的字段：`Transfer-Encoding`。简单点说，比如当客户端发送数据给服务器时，body数据的长度是在消息头指定了的。也就是通过这个字段：`Content-Length: 1024`。服务器按照这个字段去读取指定的长度即可。另外一种方式告诉服务器body数据长度是直接在body中分块表示。看下面这个例子。

```
//请求行
sock.write('HTTP/1.1 200 OK\r\n');

//消息头
sock.write('Transfer-Encoding: chunked\r\n');//必须指定Transfer-Encoding: chunked
sock.write('\r\n');

//消息体
sock.write('b\r\n');//这里的b是十六进制数，也就是下面有11个字节（统计数据长度时都不计算\r\n）
sock.write('01234567890\r\n');

sock.write('5\r\n');//这里有5个字节
sock.write('12345\r\n');

sock.write('0\r\n');//最后一个数据的长度必须是0
sock.write('\r\n');

```

下面分析代码。这里主要流程是监听客户端连接，有新连接进来就分配连接给代理处理。这里只讨论http，暂时不管https。

```lua
local skynet = require "skynet"
local socket = require "skynet.socket"
local httpd = require "http.httpd"
local sockethelper = require "http.sockethelper"
local urllib = require "http.url"
local table = table
local string = string

local mode, protocol = ...
protocol = protocol or "http"

if mode == "agent" then

//响应http请求
local function response(id, write, ...)
	local ok, err = httpd.write_response(write, ...)
	if not ok then
		-- if err == sockethelper.socket_error , that means socket closed.
		skynet.error(string.format("fd = %d, %s", id, err))
	end
end


local SSLCTX_SERVER = nil
local function gen_interface(protocol, fd)
	if protocol == "http" then
		return {
			init = nil,
			close = nil,
			read = sockethelper.readfunc(fd),//这里实际上返回的是socket.read函数
			write = sockethelper.writefunc(fd),//这里返回的是socket.write函数。
		}
	elseif protocol == "https" then
		
	else
		error(string.format("Invalid protocol: %s", protocol))
	end
end

skynet.start(function()
	skynet.dispatch("lua", function (_,_,id)
		socket.start(id)
		local interface = gen_interface(protocol, id)
		if interface.init then
			interface.init()
		end
	
		-- 把客户端发送的请求解析出来。
		-- limit request body size to 8192 (you can pass nil to unlimit)
		local code, url, method, header, body = httpd.read_request(interface.read, 8192)
		if code then --code是状态码
			if code ~= 200 then
				response(id, interface.write, code)
			else
				local tmp = {}
				if header.host then
					table.insert(tmp, string.format("host: %s", header.host))
				end
				local path, query = urllib.parse(url)
				table.insert(tmp, string.format("path: %s", path))
				if query then
					local q = urllib.parse_query(query)--获取请求的参数
					for k, v in pairs(q) do
						table.insert(tmp, string.format("query: %s= %s", k,v))
					end
				end
				table.insert(tmp, "-----header----")
				for k,v in pairs(header) do
					table.insert(tmp, string.format("%s = %s",k,v))
				end
				table.insert(tmp, "-----body----\n" .. body)
				response(id, interface.write, code, table.concat(tmp,"\n"))
			end
		else
			if url == sockethelper.socket_error then
				skynet.error("socket closed")
			else
				skynet.error(url)
			end
		end
		socket.close(id)
		if interface.close then
			interface.close()
		end
	end)
end)

else

skynet.start(function()
	local agent = {}
	local protocol = "http"
	for i= 1, 20 do--初始化20个代理，来分摊压力。
		agent[i] = skynet.newservice(SERVICE_NAME, "agent", protocol)
	end
	local balance = 1
	local id = socket.listen("0.0.0.0", 8001)
	skynet.error(string.format("Listen web port 8001 protocol:%s", protocol))
	socket.start(id , function(id, addr)
		skynet.error(string.format("%s connected, pass it to agent :%08x", addr, agent[balance]))
		skynet.send(agent[balance], "lua", id)//有新连接介入，则通知一个代理处理。
		balance = balance + 1
		if balance > #agent then
			balance = 1
		end
	end)
end)

end
```
这里主要看看服务器上怎么解析http请求的。也就是 httpd.read_request 函数
```
local function readall(readbytes, bodylimit)
	local tmpline = {}
	-- 把请求行数据和消息头数据放置在tmpline
	-- 这里返回的body表示可能是不完整的。
	local body = internal.recvheader(readbytes, tmpline, "")
	if not body then
		return 413	-- Request Entity Too Large
	end
	local request = assert(tmpline[1])--tmpline的第一个位置是 请求行数据
	local method, url, httpver = request:match "^(%a+)%s+(.-)%s+HTTP/([%d%.]+)$"
	assert(method and url and httpver)
	httpver = assert(tonumber(httpver))
	if httpver < 1.0 or httpver > 1.1 then
		return 505	-- HTTP Version not supported
	end
	local header = internal.parseheader(tmpline,2,{})--tmpline从2个位置起，都是消息头数据。
	if not header then
		return 400	-- Bad request
	end
	local length = header["content-length"]
	if length then
		length = tonumber(length)
	end
	local mode = header["transfer-encoding"]
	if mode then 
		if mode ~= "identity" and mode ~= "chunked" then
			return 501	-- Not Implemented
		end
	end

	if mode == "chunked" then --注意这个模式
		body, header = internal.recvchunkedbody(readbytes, bodylimit, header, body)
		if not body then
			return 413
		end
	else
		-- identity mode
		if length then
			if bodylimit and length > bodylimit then --数据长度超过限制
				return 413
			end
			if #body >= length then -- 实际接收的数据 大于 header["content-length"] ，则截取数据。
				body = body:sub(1,length)
			else -- 如果实际获得的数据长度 比 header["content-length"] 少，会挂起等待。
				local padding = readbytes(length - #body) 
				body = body .. padding
			end
		end
	end

	return 200, url, method, header, body
end

function httpd.read_request(...)
	local ok, code, url, method, header, body = pcall(readall, ...)
	if ok then
		return code, url, method, header, body
	else
		return nil, code
	end
end
```
我们可以继续查看请求数据是如何被解析出来的。但是主要方法是通过 `\r\n` `\r\n\r\n`来做分割数据的。我们再看看
```
local body = internal.recvheader(readbytes, tmpline, "")

function M.recvheader(readbytes, lines, header)
	if #header >= 2 then
		if header:find "^\r\n" then
			return header:sub(3)
		end
	end
	local result
	local e = header:find("\r\n\r\n", 1, true)
	if e then
		result = header:sub(e+4)
	else
		while true do
			local bytes = readbytes()
			header = header .. bytes
			e = header:find("\r\n\r\n", -#bytes-3, true) --消息头和消息体的分割字符是 "\r\n\r\n"
			if e then
				result = header:sub(e+4) --返回消息体。这里可能是完整的消息体，也可能是不完整的。
				break
			end
			if header:find "^\r\n" then
				return header:sub(3)
			end
			if #header > LIMIT then
				return
			end
		end
	end
	for v in header:gmatch("(.-)\r\n") do --"\r\n"把消息头分割成一个个的键值对
		if v == "" then
			break
		end
		table.insert(lines, v)
	end
	return result
end

```
大致过程已经了解。

#### 实际操作

在浏览器输入web服务器的ip和port，参数像下面这样。
```
http://ip:port/?color=blue&text=abc
```

返回给浏览器的结果如下：

![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190626190420.png)

#### 总结
主要是解析消息头和消息体数据
### 结束