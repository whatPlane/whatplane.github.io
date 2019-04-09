---
layout:     post                    # 使用的布局（不需要改）
title:      skynet_handle              # 标题 
subtitle:     #副标题
date:       2018-3-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - skynet源码分析
---
#### skynet_handle 
主要源码在skynet_handle.c里面。skynet_handle主要是确定了 一个 类型为int32 的数字和服务 skynet_context 的对应关系。最直观的感受，比如可以通过一个 id可以找到对应的服务，继而进行后续的操作。

```
struct handle_storage {
	struct rwlock lock;

	uint32_t harbor; //代表这个skynet节点的标志
	uint32_t handle_index;
	int slot_size;
	struct skynet_context ** slot;//存放服务的数组
	
	int name_cap;
	int name_count;
	struct handle_name *name;//handle和名字的映射
};

skynet_handle_init(int harbor) ｛

	struct handle_storage * s = skynet_malloc(sizeof(*H));
｝

```
这里主要是初始化了 handle_storage 。 可以说是仓库吧。里面有一个数组slot，预先分配了固定数量的服务skynet_context。当我们新产生一个  skynet_context 时，就会分配一个slot槽位，安放你的服务。当然这个槽位是不会直接返回给你的，而是经过处理后返回的一个 所谓的 handle 号。高八位是 harbor值，也就是代表这个节点的唯一数字。显然通过这个 handle 值，你就可以找到对应的skynet_context了
```
skynet_handle_register(struct skynet_context *ctx) {
......
for (;;) {
		int i;
		for (i=0;i<s->slot_size;i++) {
			uint32_t handle = (i+s->handle_index) & HANDLE_MASK;
			int hash = handle & (s->slot_size-1);
			if (s->slot[hash] == NULL) {
				s->slot[hash] = ctx;
				s->handle_index = handle + 1;

				rwlock_wunlock(&s->lock);

				handle |= s->harbor;
				return handle;
			}
		}
.......
```
这个 handle_storage 里面还有一个重要成员，handle_name。他提供了名字和handle的映射关系。这样通过名字也可以找到对应的 skynet_context 了。注意这里这个映射的建立和查找方法有点类似二分查找算法。下面是插入过程。
```
_insert_name(struct handle_storage *s, const char * name, uint32_t handle) {
	int begin = 0;
	int end = s->name_count - 1;
	while (begin<=end) {
		int mid = (begin+end)/2;
		struct handle_name *n = &s->name[mid];
		int c = strcmp(n->name, name);
		if (c==0) {
			return NULL;
		}
		if (c<0) {
			begin = mid + 1;
		} else {
			end = mid - 1;
		}
	}
	char * result = skynet_strdup(name);

	_insert_name_before(s, result, handle, begin);

	return result;
}
```

### 结束