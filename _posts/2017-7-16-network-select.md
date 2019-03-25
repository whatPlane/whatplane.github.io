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
  - 设为为null 则表示没有等到事件发生就一直阻塞等待，不返回
  - 如果 timeout->tv_sec=0 && timeout->tv_usec=0 ,则表示设置select成非阻塞
  - 如果 timeout的成员有一个设置为不等于0，则设置超时等待，xxxx

## 结束