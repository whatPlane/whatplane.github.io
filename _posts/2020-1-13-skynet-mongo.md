---
layout:     post                    # 使用的布局（不需要改）
title:      skynet-mongo              # 标题 
subtitle:     #副标题
date:       2020-1-13              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
skynet的跟mongo的通讯是通过sockchannel实现的。而且是工作在  seesion 模式。

### 结束