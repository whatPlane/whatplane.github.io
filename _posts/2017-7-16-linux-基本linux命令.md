---
layout:     post                    # 使用的布局（不需要改）
title:      基本linux命令               # 标题 
subtitle:     #副标题
date:       2016-7-16              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - linux
---

#### 查找文件

- find 目录 条件 [处理命令]

```
条件  可以是按照 名字 时间 文件类型等等。
处理命令 默认就是打印出来。

//查找当前目录下 m开头的文件（不递归查找，默认是递归查找的，所以加上 -maxdepth 1 ）
find . -maxdepth 1 -name "m*"

```

> `echo find -name m*`这句可以查看shell解析的样子。所以 `m*` 必须要带上 引号才能正确的表达我们的意思 





#### 查找字符串
在指定文件中查看指定字符串
- grep 字符串 文件
- grep -i 字符串 文件 // -i 表示忽悠大小写 

```
[root@instance-xwv67 tmp]# grep -i Dbus /etc/passwd
dbus:x:81:81:System message bus:/:/sbin/nologin

```
- grep -v 字符串 文件 // 查找不包含 字符串的行

```
这里演示不包含 undo 字符串的行

[root@instance-xa7uwv67 tmp]# cat -n test.log 
     1	this is a log file for test vi
     2	undo operation is:
     3	this is xxxxhhiiih,,,hhhhhhhh
     4	end
[root@instance-xa7uwv67 tmp]# grep -vn undo test.log  //这里的 n 表示显示行号
1:this is a log file for test vi
3:this is xxxxhhiiih,,,hhhhhhhh
4:end
[root@instance-xa7uwv67 tmp]# 

```
#### 查看文件

- cat 文件
- cat -n 文件 //这里会显示行号
适合查看文件内容少的

- more
每次只显示一屏幕，通过 空格 或者 回车键 可以看下一屏幕的内容 
  
- less
less的作用与more十分相似，不同点为less命令允许用户向前或向后浏览文件(PageUp键向上翻页，PageDown键向下翻页)，而more命令只能向前浏览。也就是more只能有一个方向。

- head 
有时候指向看文件开头的几行 head -5 表示查看开头的5行
tail
对应有时候只想看文件结尾的几行。

#### 统计
- wc 
返回： 行数 单词数 字节数
note：这里的单词是通过空格或者换行来作为分割的  如果显示行数 可以 wc -l

#### 查看磁盘
#### 查看内存
#### 查看cpu

## 结束