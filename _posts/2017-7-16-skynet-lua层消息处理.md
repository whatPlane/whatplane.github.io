---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-lua层消息处理              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
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
#### lua层服务的启动
我们的lua层的服务是怎么启动的？回调函数怎么注册的？
#### lua层消息的处理
skynet内部稍微封装了协程，提供了自己的几个协成专用函数。


### 结束