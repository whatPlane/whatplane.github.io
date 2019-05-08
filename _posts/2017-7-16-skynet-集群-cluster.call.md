---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-集群cluster.call              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
集群中发送消息有两种方式，一种是通过代理，一种是直接使用cluster.send cluster.call。但最终都是调用clustersender发送消息。当然消息发送又分两种类型，一种是推送，一种是请求。
#### cluster.send
看代码
```
function cluster.send(node, address, ...)
	-- push is the same with req, but no response
	local s = sender[node]
	if not s then//还没有对应的clusersender可用
		table.insert(task_queue[node], table.pack(address, ...))
	else
		skynet.send(sender[node], "lua", "push", address, skynet.pack(...))
	end
end

local sender = {}
local task_queue = {}

local function request_sender(q, node)
	local ok, c = pcall(skynet.call, clusterd, "lua", "sender", node)//获取clustersender
	if not ok then
		skynet.error(c)
		c = nil
	end
	-- run tasks in queue
	local confirm = coroutine.running()
	q.confirm = confirm
	q.sender = c
	for _, task in ipairs(q) do
		if type(task) == "table" then//这里是推送
			if c then
				skynet.send(c, "lua", "push", task[1], skynet.pack(table.unpack(task,2,task.n)))
			end
		else//这里是发送请求
			skynet.wakeup(task)
			skynet.wait(confirm)//这里等待cluster.call的调用协程唤醒
		end
	end
	task_queue[node] = nil
	sender[node] = c
end

local function get_queue(t, node)
	local q = {}
	t[node] = q
	skynet.fork(request_sender, q, node)
	return q
end

setmetatable(task_queue, { __index = get_queue } )//设置元表
```
一个节点对应一个队列，里面存放发送任务。然后fork一个协程，挂起当前cluster.send所在的协程。之后fork出来的协程把发送任务一个个的完成，也即一个个的推送消息。这里为什么会有队列，因为会有多个服务都想给外部节点推送消息，而且此时当前节点跟外部节点的通道还没有建立，所以为了保证基本的顺序（因为超过32k的长数据发送是通过c底层的低优先级链表发送，所以长数据在lua层先发送，在底层不一定真的被先发送），先把任务一个个排队保存。
#### cluster.call

看代码
```
local function get_sender(node)
	local s = sender[node]
	if not s then//还没有对应的clusersender可用
		local q = task_queue[node]
		local task = coroutine.running()
		table.insert(q, task)
		skynet.wait(task)//挂起
		skynet.wakeup(q.confirm)//唤醒队列的下一个
		return q.sender
	end
	return s
end

function cluster.call(node, address, ...)
	-- skynet.pack(...) will free by cluster.core.packrequest
	return skynet.call(get_sender(node), "lua", "req",  address, skynet.pack(...))
end
```
这里跟cluster.send类似。如果已经有建立连接了，那么直接发送消息给clustersender。如果还没有建立，那么请求获得clustersender的任务先排队，并且挂起等待。当任务完成时，再唤醒队列的下一个。

#### 总结
这里主要讨论了处理通道还没有建立，但是多个请求clustersender已经到来的问题。cluster.send因为是推送，所以不需要反复唤醒等待的过程。
### 结束