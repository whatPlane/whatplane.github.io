---
layout:     post                    # 使用的布局（不需要改）
title:      shutdown和close              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 网络
---
两个主机利用socket通讯的内核缓冲区图
![](https://gitee.com/whatplane/resource/raw/master/img/wx_20190325112547.png)
当调用close的时会同时关闭输入和输出两个方向。但是有时候我们只告诉对方：我这边已经没有数据需要发送给你了，但是你有数据还是可以继续发送给我。也就是只关闭一个方向上的数据传送。这样的场景就需要用到shutdown了。shutdown的解释如下：

```
int shutdown(int sockfd,int how);

how的方式有三种分别是

SHUT_RD（0）：关闭sockfd上的读功能，此选项将不允许sockfd进行读操作。

SHUT_WR（1）：关闭sockfd的写功能，此选项将不允许sockfd进行写操作。

SHUT_RDWR（2）：关闭sockfd的读写功能。
```

所以可以这样使用

```
shutdown(fd,SHUT_WR)
read(fd,buff,size) //继续读取对方发送的数据
```
有几点需要注意：
- shutdown关闭写就会发送fin片段给对方。
- 关闭输入缓冲区，系统会立即清除输入缓冲区的内容。
- 关闭输出缓冲区，系统会继续发送输出缓冲区剩余的内容。

## 结束