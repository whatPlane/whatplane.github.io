---
layout:     post                    # 使用的布局（不需要改）
title:      skyent-module              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
#### skynet的模块
在skynet_start.c 里面首先初始化模块单元 `M`
```
skynet_module_init(config->module_path);
```


```
skynet_module_init(const char *path) {
	struct modules *m = skynet_malloc(sizeof(*m));
	m->count = 0;
	m->path = skynet_strdup(path);

	SPIN_INIT(m)

	M = m;
}
```

M 里面主要是初始化一个指定大小的数组。每一个类型都是 skynet_module
```
struct modules {
	int count;
	struct spinlock lock;
	const char * path;
	struct skynet_module m[MAX_MODULE_TYPE];
};
```

skynet启动的时候，首先会产生一个日志服务，即skynet_context
```
struct skynet_context *ctx = skynet_context_new(config->logservice, config->logger);
```

一个服务的成员主要有 skynet_module 和 skynet_module对应的实例

```
skynet_context_new(const char * name, const char *param) {
	struct skynet_module * mod = skynet_module_query(name);

	void *inst = skynet_module_instance_create(mod);//产生实例
	
	struct skynet_context * ctx = skynet_malloc(sizeof(*ctx));
	
	ctx->mod = mod;//设置模块
	ctx->instance = inst;//设置实例
	......

```

M会分配一个索引给新的 skynet_module 。skynet_module 通过动态库所在的路径，打开动态库文件，读取其中的四个函数指针填充自己的成员。如果这个 skynet_module 是已经初始化的，那么就不用再次打开动态库文件，进行初始化了。 

```
struct skynet_module {
	const char * name;
	void * module;
	skynet_dl_create create;
	skynet_dl_init init;
	skynet_dl_release release;
	skynet_dl_signal signal;
};
```

之后通过 skynet_module 产生一个实例。这样 服务 的两个重要成员就完成了初始化。下面是日志模块的主要几个函数。注意跟上面对应。这里面实例化产生的对象实际上就是 logger 对象
```
struct logger {
	FILE * handle;
	char * filename;
	int close;
};

logger_create(void) 
logger_release(struct logger * inst) 
logger_init(struct logger * inst, struct skynet_context *ctx, const char * parm)
```


### 结束