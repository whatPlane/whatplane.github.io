---
layout:     post                    # 使用的布局（不需要改）
title:      bare参数              # 标题 
subtitle:     #副标题
date:       2015-5-20              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - git
---

#### 裸仓库
我们在初始化一个仓库一般可以使用
```
git init
git init --bare
```
加一个 --bare 表示初始化为裸仓库。裸仓库是不会显示工作区的。一般我们创建一个仓库，主要会有两部分，一个是.git目录，一个是工作区目录树。.git目录里面包括历史记录和一些git系统数据。裸仓库显示的内容就只有.git目录，并且不能使用git操作。比如我想修改最近一次的 commit message ，就会提示不行。
```
$ git commit --amend
fatal: This operation must be run in a work tree

```
当然一些只读的操作还是可以的。比如查看历史提交版本记录.

#### 使用场景
比如我在a服务器上新建了一个仓库，其他同学从a服务器clone代码到自己的本地，然后协同工作。我不希望有人直接在a服务器上修改这个仓库，那么就可以用上 --bare选项。因为如果a服务器上新建的不是裸仓库，那么以下流程就会有冲突(假设每个协作者都是独立分支)
- 王同学pull代码后修改了
- 有人直接在a服务器上修改了王同学的分支内容
- 王同学提交代码，发现冲突。
- 王同学解决冲突，再提交。

#### 总结
不过本质上多个合作总会有冲突要解决的。如果是每个协作者都是在自己的分支上开发，那么冲突可以避免一些，如果共用一个分支，那么这个裸仓库感觉也没什么很大意义。
#### 结束