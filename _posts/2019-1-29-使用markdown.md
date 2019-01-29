---
layout:     post                    # 使用的布局（不需要改）
title:      使用markdown               # 标题 
subtitle:     #副标题
date:       2019-1-29              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - tool
    - 2019
---

# 编写的md文件最后展示在网页上的样子如下： 
***

# 大小不同的标题
## 大小不同的标题
### 大小不同的标题


## 展示引用 ##
>这是markown工具的介绍 --鲁迅


## 加粗字体 ##
**MarkdownPad**  is a full-featured Markdown editor for Windows.


## 斜体字 ##
*这是斜体字* 而你现在看到的是普通字



## 无序列表 ##
- 基本使用
- 基本使用2


## 有序列表 ##
1. 第一步
	- one step
	- two step
2. 第二步
	- one step
	- two step
3. 第三部


## 插入网址 ##
Don't guess if your [hyperlink syntax](http://markdownpad.com) is correct; LivePreview will show you exactly what your document looks like every time you press a key.


## 插入图片 ##
![](https://gitee.com/whatplane/resource/raw/master/img/set.png) 
> 我的图片资源都放置在gitchina的仓库里面

## 分割线 ##
***


## 演示怎么插入代码
```
skynet.start(function()
	skynet.newservice ("debug_console", config.debug_port)
	skynet.newservice ("protod")
	skynet.uniqueservice ("database")

	local loginserver = skynet.newservice ("loginserver")
	skynet.call (loginserver, "lua", "open", login_config)	

	local gamed = skynet.newservice ("gamed", loginserver)
	skynet.call (gamed, "lua", "open", game_config)
end)
```


# md源文件主要内容如下：  
***

```
# 大小不同的标题
## 大小不同的标题
### 大小不同的标题

## 展示引用 ##
>这是markown工具的介绍 --鲁迅

## 加粗字体 ##

**MarkdownPad**  is a full-featured Markdown editor for Windows.


## 斜体字 ##
* 这是斜体字 * 而你现在看到的是普通字

## 无序列表 ##

- 基本使用
- 基本使用2


## 有序列表 ##
1. 第一步
	- one step
	- two step
2. 第二步
	- one step
	- two step
3. 第三部

## 插入网址 ##
Don't guess if your [hyperlink syntax](http://markdownpad.com) is correct; LivePreview will show you exactly what your document looks like every time you press a key.

## 插入图片 ##
![](https://gitee.com/whatplane/resource/raw/master/img/set.png) 

## 分割线 ##
***    
```
## 插入代码的方法
![](https://gitee.com/whatplane/resource/raw/master/img/markdown-1.png)
> 字符 ` 的在键盘的位置如下：

![](https://gitee.com/whatplane/resource/raw/master/img/markdown-2.jpg)
