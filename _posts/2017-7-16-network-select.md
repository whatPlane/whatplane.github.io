---
layout:     post                    # 使用的布局（不需要改）
title:      select              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 网络
---

## select
函数介绍
```
struct timeval｛

long tv_sec;//秒数

long tv_usec;//微秒数

｝;

int select(int nfds, fd_set *readfds, fd_set *writefds,fd_set *exceptfds,
 			struct timeval *timeout);
```
- nfds 表示监听的描述符集合中最大值再加1， 即 maxfd + 1
- readfds writefds exceptfds 是分别需要监听的三个集合。设置为null则表示不关注。
- timeout是超时设置。
  - 设为为null 则表示一直阻塞等待，知道有事件发生。
  - 如果 timeout->tv_sec=0 && timeout->tv_usec=0 ,则表示设置select成非阻塞
  - 如果 timeout->tv_sec！=0 || timeout->tv_usec!=0，则设置超时等待，如果到超时事件还没有事件发生则返回0
  - 有事件发生，则返回值已经准备好的描述符个数。

## fd_set
fd_set可以看做是一个位集合。可以把对应的位设置为1和0.这个位跟描述符是对应的。观察下图


使用select模型的流程和注意点

## 结束