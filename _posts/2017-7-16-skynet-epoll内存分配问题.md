---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-epoll内存分配问题              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
之前讨论基本上没有仔细分析内存的分配和释放。这里从三个方面来分别讨论。
##### 本地服务间的通讯
我们服务间发送信息一般这样调用skyent.send
```
function skynet.send(addr, typename, ...)
	local p = proto[typename]
	return c.send(addr, p.id, 0 , p.pack(...))
end
```
这里p.pack实际上是 skynet.pack。这里可以看[skynet.pack](https://whatplane.github.io/2018/03/17/skynet-skynet.pack/).这里把lua数据转化成了 pointer + size。也就是对应在底层分配了一块内存。最终会在目的服务的队列中push一个消息。而消息的data就是pointer。当我们的目的服务接收到消息时，会通过skynet.unpack把 pointer+ size解包出来，得到lua数据类型。这个时候并没有把pointer指向的内存释放，而是等待lua层消息处理完成后，底层再把pointer指向的内存释放。
![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190424185951.png)
##### 网络消息的接收-gate服务
这里主要是处理了首部为2字节的流数据。gate服务收到的网络消息skynet_socket_message格式如下 
![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190424193040.png)
gate通过过滤 **网络数据** 提取网络包。具体是指把buffer里面的数据copy一份保存到队列，然后释放buffer。也就是说队列里面是有网络数据的。当gate收到网络数据时，转发有两种方式，看下面的代码

```
function handler.message(fd, msg, sz)
	-- recv a package, forward it
	local c = connection[fd]
	local agent = c.agent
	if agent then//直接转发到agent
		skynet.redirect(agent, c.client, "client", 1, msg, sz)
	else//转发给watchdog
		skynet.send(watchdog, "lua", "socket", "data", fd, netpack.tostring(msg, sz))
	end
end
```

- 转发给agent


看代码skynet.redirect

```
skynet.redirect = function(dest,source,typename,...)
	return c.redirect(dest, source, proto[typename].id, ...)
end

static int
lredirect(lua_State *L) {
	uint32_t source = (uint32_t)luaL_checkinteger(L,2);
	return send_message(L, source, 3);
}

send_message(lua_State *L, int source, int idx_type) {
		...
		void * msg = lua_touserdata(L,idx_type+2);
		int size = luaL_checkinteger(L,idx_type+3);
		if (dest_string) {
			session = skynet_sendname(context, source, dest_string, type | PTYPE_TAG_DONTCOPY, session, msg, size);
		} else {
			session = skynet_send(context, source, dest, type | PTYPE_TAG_DONTCOPY, session, msg, size);
		}
		break;
```

也就是说这里吧 网络包 直接push到agent对应的服务队列里面了。至此gate的处理完成了，底层把skyent_socket_message释放了。但是没有释放 网络包。但是agent在处理网络包后，把网络包释放了。

- 转发给watchdog


注意gate转发给watchdog的代码


```
skynet.send(watchdog, "lua", "socket", "data", fd, netpack.tostring(msg, sz))

static int
ltostring(lua_State *L) {
	void * ptr = lua_touserdata(L, 1);
	int size = luaL_checkinteger(L, 2);
	if (ptr == NULL) {
		lua_pushliteral(L, "");
	} else {
		lua_pushlstring(L, (const char *)ptr, size);//转化成lua字符串
		skynet_free(ptr);//释放分配的内存
	}
	return 1;
}

```

通过netpack.tostring(msg, sz)把底层数据被转化成lua字符串，同时释放底层分配的内存。当然skynet.send最后发送时又把lua数据打包成 pointer +size 了。但是这已经是一个新的发送过程了。
##### 网络消息的接收-普通sock
这里讨论的是不过滤的网络处理。也就是[sock.lua的实现](https://whatplane.github.io/2018/03/17/skynet-lua%E5%B1%82%E8%AF%BB%E5%86%99%E6%95%B0%E6%8D%AE/).这里对应每个sock都会分配一个内存池。收到数据的时候，找到一个空闲块，然后把网络数据挂载到他身上。而读数据，就是把已经使用的块上挂载的网络数据取出（这里是把网络数据pointer+size转化为lua字符串，然后释放pointer）。最后服务销毁的时候，整个内存池也销毁。
##### 总结
本地服务间的消息都是发送方分配内存，接收方处理完后，释放。注意网络消息最终也是挂载在本地消息上的，但是消息处理后，只是释放了skynet_socket_message分配的内存，其buffer分配的内存还需要另外释放。
![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190425114500-min.png)

### 结束