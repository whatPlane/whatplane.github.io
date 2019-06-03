---
layout:     post                    # 使用的布局（不需要改）
title:      skynet的makefile              # 标题 
subtitle:     #副标题
date:       2017-7-16              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - makefile
---
首先我们要清楚在linux系统上的构建命令是
```
make linux
```
再看skynet 的Makefile 的一行是
```
include platform.mk

```
那我们先分析 platform.mk
```
PLAT ?= none # 如果PLAT没有值，那么就赋值为none。 ？= 可以认为就是设置默认值
PLATS = linux freebsd macosx

CC ?= gcc

.PHONY : none $(PLATS) clean all cleanall # 定义几个伪目标

#ifneq ($(PLAT), none) #比较语句

.PHONY : default

default :
	$(MAKE) $(PLAT)

#endif

none :
	@echo "Please do 'make PLATFORM' where PLATFORM is one of these:"
	@echo "   $(PLATS)"

SKYNET_LIBS := -lpthread -lm # 一些编译选项设置
SHARED := -fPIC --shared
EXPORT := -Wl,-E

linux : PLAT = linux # 在依赖里面执行赋值操作 PLAT = linux
macosx : PLAT = macosx
freebsd : PLAT = freebsd

macosx : SHARED := -fPIC -dynamiclib -Wl,-undefined,dynamic_lookup
macosx : EXPORT :=
macosx linux : SKYNET_LIBS += -ldl
linux freebsd : SKYNET_LIBS += -lrt

# Turn off jemalloc and malloc hook on macosx

macosx : MALLOC_STATICLIB :=
macosx : SKYNET_DEFINES :=-DNOUSE_JEMALLOC

linux macosx freebsd : #任意目标都会执行下面的命令 
	$(MAKE) all PLAT=$@ SKYNET_LIBS="$(SKYNET_LIBS)" SHARED="$(SHARED)" EXPORT="$(EXPORT)" MALLOC_STATICLIB="$(MALLOC_STATICLIB)" SKYNET_DEFINES="$(SKYNET_DEFINES)"

```
注意一点知识
- $@ 表示目标
- $< 表示第一个依赖 
- $^ 表示所有依赖


再看完整的Makefile
```
include platform.mk

LUA_CLIB_PATH ?= luaclib # skynet中 lua_clib库存放的路径。包括第三方的和skynet自己的
CSERVICE_PATH ?= cservice # 几个基本so ，比如snlua log 的存放路径

SKYNET_BUILD_PATH ?= . #构建目录就是当前目录

CFLAGS = -g -O2 -Wall -I$(LUA_INC) $(MYCFLAGS) 
# CFLAGS += -DUSE_PTHREAD_LOCK

# lua

LUA_STATICLIB := 3rd/lua/liblua.a # 设置即将生产的lua5.3静态库的路径和名字
LUA_LIB ?= $(LUA_STATICLIB)
LUA_INC ?= 3rd/lua

$(LUA_STATICLIB) :
	cd 3rd/lua && $(MAKE) CC='$(CC) -std=gnu99' $(PLAT)

# https : turn on TLS_MODULE to add https support

# TLS_MODULE=ltls
TLS_LIB=
TLS_INC=

# jemalloc

JEMALLOC_STATICLIB := 3rd/jemalloc/lib/libjemalloc_pic.a #设置后面打算生产的jemallogc静态库的路径和名称
JEMALLOC_INC := 3rd/jemalloc/include/jemalloc

all : jemalloc
	
.PHONY : jemalloc update3rd

MALLOC_STATICLIB := $(JEMALLOC_STATICLIB)

$(JEMALLOC_STATICLIB) : 3rd/jemalloc/Makefile
	cd 3rd/jemalloc && $(MAKE) CC=$(CC) 

3rd/jemalloc/autogen.sh :
	git submodule update --init 

3rd/jemalloc/Makefile : | 3rd/jemalloc/autogen.sh
	cd 3rd/jemalloc && ./autogen.sh --with-jemalloc-prefix=je_ --enable-prof

jemalloc : $(MALLOC_STATICLIB)

update3rd : 
	rm -rf 3rd/jemalloc && git submodule update --init

# skynet

CSERVICE = snlua logger gate harbor
LUA_CLIB = skynet \
  client \
  bson md5 sproto lpeg $(TLS_MODULE)

LUA_CLIB_SKYNET = \
  lua-skynet.c lua-seri.c \
  lua-socket.c \
  lua-mongo.c \
  lua-netpack.c \
  lua-memory.c \
  lua-profile.c \
  lua-multicast.c \
  lua-cluster.c \
  lua-crypt.c lsha1.c \
  lua-sharedata.c \
  lua-stm.c \
  lua-debugchannel.c \
  lua-datasheet.c \
  \

SKYNET_SRC = skynet_main.c skynet_handle.c skynet_module.c skynet_mq.c \
  skynet_server.c skynet_start.c skynet_timer.c skynet_error.c \
  skynet_harbor.c skynet_env.c skynet_monitor.c skynet_socket.c socket_server.c \
  malloc_hook.c skynet_daemon.c skynet_log.c

all : \
  $(SKYNET_BUILD_PATH)/skynet \
  $(foreach v, $(CSERVICE), $(CSERVICE_PATH)/$(v).so) \
  $(foreach v, $(LUA_CLIB), $(LUA_CLIB_PATH)/$(v).so) 

$(SKYNET_BUILD_PATH)/skynet : $(foreach v, $(SKYNET_SRC), skynet-src/$(v)) $(LUA_LIB) $(MALLOC_STATICLIB)
	$(CC) $(CFLAGS) -o $@ $^ -Iskynet-src -I$(JEMALLOC_INC) $(LDFLAGS) $(EXPORT) $(SKYNET_LIBS) $(SKYNET_DEFINES)

$(LUA_CLIB_PATH) :
	mkdir $(LUA_CLIB_PATH)

$(CSERVICE_PATH) :
	mkdir $(CSERVICE_PATH)

define CSERVICE_TEMP
  $$(CSERVICE_PATH)/$(1).so : service-src/service_$(1).c | $$(CSERVICE_PATH)
	$$(CC) $$(CFLAGS) $$(SHARED) $$< -o $$@ -Iskynet-src
endef

$(foreach v, $(CSERVICE), $(eval $(call CSERVICE_TEMP,$(v))))

$(LUA_CLIB_PATH)/skynet.so : $(addprefix lualib-src/,$(LUA_CLIB_SKYNET)) | $(LUA_CLIB_PATH)
	$(CC) $(CFLAGS) $(SHARED) $^ -o $@ -Iskynet-src -Iservice-src -Ilualib-src

$(LUA_CLIB_PATH)/bson.so : lualib-src/lua-bson.c | $(LUA_CLIB_PATH)
	$(CC) $(CFLAGS) $(SHARED) -Iskynet-src $^ -o $@ -Iskynet-src

$(LUA_CLIB_PATH)/md5.so : 3rd/lua-md5/md5.c 3rd/lua-md5/md5lib.c 3rd/lua-md5/compat-5.2.c | $(LUA_CLIB_PATH)
	$(CC) $(CFLAGS) $(SHARED) -I3rd/lua-md5 $^ -o $@ 

$(LUA_CLIB_PATH)/client.so : lualib-src/lua-clientsocket.c lualib-src/lua-crypt.c lualib-src/lsha1.c | $(LUA_CLIB_PATH)
	$(CC) $(CFLAGS) $(SHARED) $^ -o $@ -lpthread

$(LUA_CLIB_PATH)/sproto.so : lualib-src/sproto/sproto.c lualib-src/sproto/lsproto.c | $(LUA_CLIB_PATH)
	$(CC) $(CFLAGS) $(SHARED) -Ilualib-src/sproto $^ -o $@ 

$(LUA_CLIB_PATH)/ltls.so : lualib-src/ltls.c | $(LUA_CLIB_PATH)
	$(CC) $(CFLAGS) $(SHARED) -Iskynet-src -L$(TLS_LIB) -I$(TLS_INC) $^ -o $@ -lssl

$(LUA_CLIB_PATH)/lpeg.so : 3rd/lpeg/lpcap.c 3rd/lpeg/lpcode.c 3rd/lpeg/lpprint.c 3rd/lpeg/lptree.c 3rd/lpeg/lpvm.c | $(LUA_CLIB_PATH)
	$(CC) $(CFLAGS) $(SHARED) -I3rd/lpeg $^ -o $@ 

clean :
	rm -f $(SKYNET_BUILD_PATH)/skynet $(CSERVICE_PATH)/*.so $(LUA_CLIB_PATH)/*.so

cleanall: clean
ifneq (,$(wildcard 3rd/jemalloc/Makefile))
	cd 3rd/jemalloc && $(MAKE) clean && rm Makefile
endif
	cd 3rd/lua && $(MAKE) clean
	rm -f $(LUA_STATICLIB)


```
上面这段代码主要需要清楚的是，skynet的库的划分。主要分为三类。
- skynet-src里面的c代码构建出一个可执行文件skynet。也就是后面我们执行`skynet conf`时用到的skynet。
- 第二部分是lua_clib库。这里又分为skynet自身的lua_clib库和第三方lua_clib库
  - lualib-src的c代码构建出来的动态库skyent.so bson.so
  - 第三方lua_clib的c代码构建出来的动态库,比如 md5.so lpeg.so
- 第三部分是service-src的c代码构建出的动态库 snlua.so logger.so等。

> `$(foreach v, $(CSERVICE), $(CSERVICE_PATH)/$(v).so)`产生cservice下的so库。 $(foreach v,$list,fun(v))意思是是把list中的元素一个个取出后，进行fun处理。

#### 总结

#### 结束


