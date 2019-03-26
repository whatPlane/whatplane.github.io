---
layout:     post                    # 使用的布局（不需要改）
title:      MTU和mss              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 网络
---
![](https://gitee.com/whatplane/resource/raw/master/img/2843224-029d06502bd58f0e.png)
## MTU
mtu是Maxitum Transmission Unit 即最大传输单元。他指的是链路层的frame的负载的最大值。负载的范围是46-1500个字节。如果这个负载超过mtu，那么就会造成ip分片。ip协议中有专门标识ip分片的标志位。当一个路由器收到一个frame，发现他大鱼mtu时，则可能会把该ip包分解为更小的ip包，再次转发，当然可能会直接丢弃。
## mss
mss是Maxitum Segment Size 即最大分段大小。这个主要是为tcp协议服务的。我们在tcp三次握手的时候，会发现mss字段，也就是告诉对方我能接收的最大segment的大小。这个mss是根据mtu来的。一般为1500-20-20=1460.因为ip头和tcp头最小的大小是20+20字节。也就是tcp发送的片段不能超过1460，否则就会被分成多个ip包。而tcp是以一个tcp分片就是一个ip包
来控制实现重发，有序和流量控制的。这样超过mss显然就会出问题。

## 结束