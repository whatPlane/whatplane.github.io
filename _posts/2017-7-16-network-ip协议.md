---
layout:     post                    # 使用的布局（不需要改）
title:      ip协议              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 网络
---

## ip协议的数据报分析
这里是ipv4的。主要分为数据报head和body
![](https://gitee.com/whatplane/resource/raw/master/img/jt_20190315105601-min.png)
有几个重点关注的位置：
- 红色是最重要的，也就是源地址和目的地址
- totoal length 表示这个数据报的总长度
- ihl 表示数据报head的长度。data上面是head部分。我们看到data上面有个options字段，这个是导致head的大小不确定的因素。ihl占用4个位。也就是最大值是15，表示header的字节是15x4。**也就是说这里的单位是4字节**。即ihl如果是2则表示header大小是8字节。
- time to live 表示这个ip包的生命周期。比如说设置为3，那么没经过一个路由数字就减一，不断的经过路由器，数字减小，到0的时候，ip包就会被丢弃。当然为0的时候路由器会向上游主机报告，通过icmp协议。
- Identification, flags和fragment offset 主要是为处理碎片服务。有时候ip包太大，某个子网络的mtu太小，就只能通过路由器切割成小的ip包，再转发。这个与tcp的segment是完全不同的。tcp的segment是带有序号的，主要是为了保证传输的数据流顺序不被打乱，是实现tcp协议的保证，是必要的。
- protocol 表示这个ip包的上层的（传输层）的类型。比如有icmp tcp udp等。
- header checksum 校验和。**这个主要是校验header信息**。body校验交给上层协议。接收方如果发现有异常就会丢弃。

## icmp协议
ip协议目的就是把源ip的主机信息传递到目的ip主机。但是传输过程中会出现很多问题。比如网线断开了，网络拥塞或者路由器下线了等会导致ip包无法到达。但是上游的主机并不清楚，他们可能还在继续发送下一个消息。这样消息的传递就不可靠了。icmp协议就可以向上游报告错误。比如数据传递过程中，路由器发现ip包的ttl已经为0了，那么就会发送icmp数据包上报。route显示经过的路由器路径就是通过icmp协议实现，即通过不断发包递增设置ttl，收集反馈信号，来追踪路径。icmp协议的另一个作用是查询。ping的实现就是利用icmp协议。

## 结束