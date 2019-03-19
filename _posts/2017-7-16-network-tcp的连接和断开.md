---
layout:     post                    # 使用的布局（不需要改）
title:      tcp的连接和断开             # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 网络
---


熟悉这几张图是必要的。图-1
![](https://gitee.com/whatplane/resource/raw/master/img/wx_20190319140048-min.png)
这张图主要是说明了tcp通讯的整个过程。
- 建立连接
- 数据传送。也就是我们调用read write的阶段
- 关闭连接

建立连接的过程：
- 发起方发送syn，自己进入syn_sent状态。等待对方发送syn和ack。
- 接收方收到syn后进入syn_rcvd状态，然后发送syn+ack，等待ack。
- 发起方收到syn+ack进入establish状态，然后发送ack。
- 接收方收到ack进入establish状态。

连接的细节：发起方发送syn片段，服务端收到syn片段后，放入一个半连接队列，这个队列里面的都是syn_rcvd状态。半连接队列是有固定大小的。当服务端发送syn+ack，之后长时间没有收到ack，则会进程重发。重发的次数和重发的时间总和是有限制的。如果最后超时，则会从半连接队列删除记录。正常状态收到ack后，则把半连接队列的内容取出来放入全连接队列。这个队全连接列的长度是min（somaxconn，backlog）。backlog就是listen函数的第二个参数。之后应用层通过accept从全连接队列里取出一个个连接。

> syn flood就是伪造客户端发起syn连接，而服务端发送syn+ack后一直无法收到ack（因为伪造的客户端根本就无法找到，也就无法返回ack），会重发消耗cpu，同时占用半连接队列，导致真正的用户连接无法成功建立。


断开连接的过程：
- 发起方发送fin片段，自己进入fin_wait1状态。
- 接收方收到fin，发送ack。同时自己进入close_wait状态。可以理解为接收方期望应用程序调用close函数发送fin片段。
- 发起方收到ack，进入fin_wait2状态。
- 接收方调用close函数发送fin片段，期望接受ack，进入last_ack。
- 发起方接收fin，进入time_wait状态。并发送ack。这里会等待2msl的时间最后进入closed状态。
- 接收方收到最后的ack后进入closed状态。


有几个地方说明一下：
- syn和fin 片段都需要占用序列号。这与单独的ack片段不占用序列号区别。
- 图中mss表示最大段大小。主要是告诉对方，我这边能够承受的最大段大小，好让对方调整发送ip包的尺寸，防止让路由器处理碎片问题。
- close函数不一定会立即发送fin片段。见 [shutdwon和close](https://whatplane.github.io/)
- 为什么接收方收到发起方的fin后，只发送一个ack，而不是ack+fin呢？发起者发送一个fin是向对方表示，我这边没有数据要发送给你了，但是还是可以继续接受数据的。而接收方可能还有数据需要发送，也就是接收方可能还在调用write写数据，最后才调用close，发送了一个fin片段。
- 发起方收到了fin后，发送了一个ack给接收方，为什么要需要等待2msl呢？原因是为了确保接收方收到了这个ack。如果ack丢失了，接收方可能重发fin，如果发起者已经closed了，那么对方将一直处于last_ack状态。



下面这张图解释了状态变化。实线表示客户端状态改变，虚线表示服务端状态改变。
![](https://gitee.com/whatplane/resource/raw/master/img/wx_20190318190757-min.png)
关于同时断开。当发起方发送fin期待，对方发送ack时，却收到了fin。这就表示双方都想断开。要想断开，本质上都要是确认收到了对方的fin。
- 发起方发送了fin后进入fin_wait1状态。此时收到fin，发送ack则进入closing状态，再收到对我发起方fin的确认ack则进入time_wait状态。
- 发起方发送fin后此事后收到了fin+ack，也就是表示对方已经收到了我发的fin。则发起方只需要发送ack，告诉对方我也收到了你的fin，即可进入time_wait状态。


对于上面的图有rst的地方不是清楚，待研究。
下面是关于同时打开的分析。
![](https://gitee.com/whatplane/resource/raw/master/img/wx_20190319115455-min.png)
同时打开的意思是：当一方发送syn，期望收到syn+ack时，却收到了对方发来的syn。
- 当发送syn后进入syn_sent状态。
- 当收到syn时进入syn_rcvd状态。**这时候双方都像是服务端**（跟上面的转化图似乎有点矛盾）。然后发送syn+ack。syn跟之前发送的一样。然后期待收到ack。这里发送出的ack就是向对方表示，我收到了你的syn片段。所以最后双方进入establish状态。



## 结束