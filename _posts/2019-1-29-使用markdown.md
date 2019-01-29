---
layout:     post                    # 使用的布局（不需要改）
title:      使用markdown               # 标题 
subtitle:   我的副标题-开始写文字了  #副标题
date:       2019-1-29              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - tool
---

## markdwon的使用演示
>这是markdown工具的介绍。

## Welcome to MarkdownPad 2 ##

**MarkdownPad** is a full-featured Markdown editor for Windows.

### Built exclusively for Markdown ###

Enjoy first-class Markdown support with easy access to  Markdown syntax and convenient keyboard shortcuts.

Give them a try:

- **Bold** (`Ctrl+B`) and *Italic* (`Ctrl+I`)
- Quotes (`Ctrl+Q`)
- Code blocks (`Ctrl+K`)
- Headings 1, 2, 3 (`Ctrl+1`, `Ctrl+2`, `Ctrl+3`)
- Lists (`Ctrl+U` and `Ctrl+Shift+O`)

### See your changes instantly with LivePreview ###

Don't guess if your [hyperlink syntax](http://markdownpad.com) is correct; LivePreview will show you exactly what your document looks like every time you press a key.

### Make it your own ###

Fonts, color schemes, layouts and stylesheets are all 100% customizable so you can turn MarkdownPad into your perfect editor.

### A robust editor for advanced Markdown users ###

MarkdownPad supports multiple Markdown processing engines, including standard Markdown, Markdown Extra (with Table support) and GitHub Flavored Markdown.

With a tabbed document interface, PDF export, a built-in image uploader, session management, spell check, auto-save, syntax highlighting and a built-in CSS management interface, there's no limit to what you can do with MarkdownPad.

***
# 学习使用markdwon #
- 基本使用
- 基本使用2


## 第二个小结 ##
1. 第一步
	- one step
	- two step
2. 第二步
	- one step
	- two step
3. 第三部

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


**hello**
>你是对的 --鲁迅


这是演示插入张图片：
![](https://gitee.com/whatplane/resource/raw/master/img/set.png) 




