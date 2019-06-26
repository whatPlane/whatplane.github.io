---
layout:     post                    # 使用的布局（不需要改）
title:      sproto              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
这个是skynet自带的一个网络协议解析工具，作用类似protobuf。一般来说，大部分协议都是客户端发起请求，服务器回应请求。按照skynet的建议，可以把客户端给服务器的请求协议保存在slot1 ，把服务器发送给客户端的请求保存在slot2。sproto的目的是
- 把我们的lua表，根据指定的协议格式`理解`后，打包成二进制数据
- 把收到的二进制数据正确解析成为lua表


```
local sprotoparser = require "sprotoparser"

local proto = {}

proto.c2s = sprotoparser.parse [[
.package {
	type 0 : integer
	session 1 : integer
}

handshake 1 {
	response {
		msg 0  : string
	}
}

get 2 {
	request {
		what 0 : string
	}
	response {
		result 0 : string
	}
}

set 3 {
	request {
		what 0 : string
		value 1 : string
	}
}

quit 4 {}

]]

proto.s2c = sprotoparser.parse [[
.package {
	type 0 : integer
	session 1 : integer
}

heartbeat 1 {}
]]

return proto

```
c2s表示客户端发起的请求，s2c表示服务器发起的请求。看这个协议内容，我们很容易发现，大部分都是c2s的内容，也就是大部分是客户端发起请求，服务器回应请求。从客户端说起。客户端需要发送数据和接受数据。发送数据的时候，需要把lua表的数据打包成二进制，然后发送；另外，接受数据完后，需要把已经收到的数据，解析成lua表，供后续业务逻辑使用。为了满足让客户端能够正确的处理数据，需要先做一些初始化处理，如下
```
--通过下面这行代码，客户端可以
--1.通过host正确解析服务器的请求数据
--2.正确打包应该回应给服务器的数据。
local host = sproto.new(proto.s2c):host "package"

-- 这样，在发送给服务器的请求时，能够正确打包数据。
local request = host:attach(sproto.new(proto.c2s)) 
```
注意：我一直强调的是 **发送请求** 。比如说服务器发送数据给客户端：可能是客户端主动发起的请求，此时服务器只是回应请求；也可能是服务器主动发起的请求。上面的协议分c2s 和s2c就是这一个意思。特别是c2s中可以看到这种
```
get 2 {
	request {
		what 0 : string
	}
	response {
		result 0 : string
	}
}

handshake 1 {
	response {
		msg 0  : string
	}
}
```
- 看上面的get协议 。客户端发起请求的时候，通过request，客户端知道如何打包发送给服务器的数据。服务器收到请求后，通过request可以清楚客户端发来的数据格式，同时通过response可以清楚自己应该如何打包即将回应给客户端的数据。  
- 再看handshake协议。这里客户端发送请求给服务器的时候，其实是没有请求数据的。因为没有request选项。但是服务器收到请求，依旧会通过handshake返回正确的数据。


再看服务器发起的请求。也就是s2c协议。
```
proto.s2c = sprotoparser.parse [[
.package {
	type 0 : integer
	session 1 : integer
}

heartbeat 1 {}
]]
```
也就是说，服务器主动发起的请求就只有一条。heartbeat。再看服务器的初始化处理
```
--这里host是通过c2s协议数据产生的.所以可以理解客户端发起的请求数据和正确打包回应请求需要的数据
host = sprotoloader.load(1):host "package"

--这里send_request函数，是绑定了s2c协议数据的。所以知道服务器发起请求时，该如何正确打包数据。
send_request = host:attach(sprotoloader.load(2))
```


#### 总结
这里可能产生理解混乱的地方是协议中的request 和 response 的真正作用。只要我们搞清楚了谁是发起请求的那一方就应该没有问题了。
#### 结束