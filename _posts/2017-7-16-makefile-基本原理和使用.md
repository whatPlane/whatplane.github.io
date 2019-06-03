---
layout:     post                    # 使用的布局（不需要改）
title:      基本原理和使用              # 标题 
subtitle:     #副标题
date:       2017-7-16              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - makefile
---


#### 基本原理
```
target:依赖1 依赖2 ...
	shell 命令
	shell 命令

依赖1: 依赖a 依赖b 。。。
	shell 命令
	shell 命令

...

```
也就是说你想执行shell命令，可以理解为你想生成一个target，那么前提是那些依赖都存在。这里面是存在层级关系的。更进一步的说是这样.你想要生成a，那么先要保证a的依赖b存在，而要生成b，要首先保证b的依赖c存在。target可以是一个文件，也可以是一个所谓的伪目标。
- 目标是文件的情况

当发现当前目录没有对应的目标文件时，一定会执行对应的命令。如果目标文件已经存在，如果依赖发生了变化，那么shell命令也会执行。问题来了，如何发现依赖发生了变化。看最简单的情况
```
app：main.c
	gcc -o app -c main.c 
 
```
也就是main.c发生了变化，那么就会执行shell命令重新生成app。如果依赖b是一眼无法发现是否被修改的，那么就会去查找以b作为目标的依赖是否发生改变。如果依赖是空也就是这样子
```
app:
	xxx # create app file
```
那么第一次肯定会生成一个a文件。当二次构建程序的时候，就不会执行shell命令了。因为依赖都没有，就没有被修改一说了。如果就是想每次构建都执行对应的shell呢？只要每次都当app目标不存在就好了。于是就有了伪目标。 

- 目标是伪目标


只需要像下面这样定义即可


```
.PHONY : xxx

xxx: 依赖
	shell命令


```

> 注意shell命令的开头是一个tab，不是空格。

#### 使用流程
一般我们都会定义一个 Makefile 或者 makefile的文件。当我们执行
```
make 目标
```
的时候，就会执行对应的命令了。
当然，如果你的makefile文件名是自定义的，你要这样
```
make -f  filename 目标
```
当没有指定目标的时候，那么默认目标就是makefile中出现的第一个目标

##### 一点补充
`:=`跟我们平时理解的赋值基本没有差别。但是`=`是有差别的。看例子
```

a=one
b=$(a)
a=two

```
这里b的值是two。也就是说a的值是在扫描完整个makefile文件后决定他的值是多少的。

#### 看个例子

![](https://gitee.com/whatplane/resource/raw/master/img/xx_20190530115706.png) 
我这里的的makefile会在output目录下生成两个执行文件 app1和app2。 00_Common 目录表示 01_Application1 和 01_Application2 都会用到。

```
NAME_APP1 = app1
NAME_APP2 = app2

TARGETS = $(NAME_APP1) $(NAME_APP2)

SOURCE_COMMON = $(wildcard ./00_Common/*.c) # 获取当前00_Common目录下的所有.c文件
SOURCE_APP1 = $(SOURCE_COMMON) $(wildcard ./01_Application1/*.c)
SOURCE_APP2 = $(SOURCE_COMMON) $(wildcard ./02_Application2/*.c)

#利用warning函数打印消息
$(warning "the value of SOURCE_APP1 is$(SOURCE_APP1)")

OBJ_APP1 = $(patsubst %.c, %.o, $(SOURCE_APP1)) # 把所有.c文件替换成.o文件
OBJ_APP2 = $(patsubst %.c, %.o, $(SOURCE_APP2))

INCLUDE_COMMON = -I./00_Common/ # 指定头文件目录

CFLAGS = -Wall -c  # 添加编译选项
MYDEBUG = -g
CC = gcc

all: $(TARGETS)
	@echo 'for create all'
$(NAME_APP1): $(OBJ_APP1)	
	@mkdir -p output/
	@echo 'create app1'
	$(CC) $(OBJ_APP1)  -o output/$(NAME_APP1)  # 产生app1可执行文件到output目录

$(NAME_APP2): $(OBJ_APP2)	
	@mkdir -p output/
	@echo 'create app2'
	$(CC) $(OBJ_APP2)  -o output/$(NAME_APP2) # 产生app2可执行文件到output目录

%.o: %.c #把.c源文件转变成.o目标文件
	@echo 'create obj'
	$(CC) $(INCLUDE_COMMON) $(MYDEBUG) $(CFLAGS) $< -o $@

.PHONY: clean
clean:
	rm -rf $(OBJ_APP1) $(OBJ_APP2) output/ #清除目标文件和output目录


```
当执行`make `时，这次的目标是all
#### 使用
#### 结束


