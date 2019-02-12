---
layout:     post                    # 使用的布局（不需要改）
title:      screen               # 标题 
subtitle:     #副标题
date:       2017-7-16              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - tool
---
# 为什么使用screen
当你在ssh terminal执行shell时，如果网络这时断开，你的程序会怎样？TERMINATED呀！有了screen，就可以让程序跑在screen而不会随着ssh的断开而断开 

# screen的使用
1.安装Screen
CentOS系统中执行：yum install screen 

2.常用命令
screen -S test    #创建一个名为test的会话
screen -ls            #列出所有会话
screen -d test    #卸载名为test的会话，但会话中的任务会继续执行。
screen -r test      #恢复名为test的会话
exit                    #退出当前窗口

![](https://gitee.com/whatplane/resource/raw/master/img/36043032.png) 
