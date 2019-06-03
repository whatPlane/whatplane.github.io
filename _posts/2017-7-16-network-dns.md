---
layout:     post                    # 使用的布局（不需要改）
title:      dns服务              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - network
---
dns服务器主要提供了域名和ip的查询。一般本机都配置了默认的dns服务器地址，我们需要访问某个域名的时候，实际上是先通过dns服务器查询域名对应的ip，然后通过ip地址访问的目标。dns的一般查询流程是这样。首先询问默认dns服务器某域名的ip地址是多少，如果查到了自然就马上返回了，如果没有查找则会想上一级dns服务器询问，如果一直查询不到，则继续上一级，直到root-dns服务器。最后root-dns服务器会直到该怎么查询，查询到结果后，把结果原路返回给发起询问的主机。这里主要涉及到一个函数
```
struct hostent *gethostbyname(const char *name);
struct hostent
    {
        char    *h_name;   官方名            
        char    **h_aliases; 多个别名
        int     h_addrtype; 是ipv4 还是 ipv6
        int     h_length; 主机ip地址的长度
        char    **h_addr_list; 里面存放多个IP地址 都是网络字节序 每个ip地址都是null结束的字符串
        #define h_addr h_addr_list[0] 方便取出第一个ip地址
    };

```
下面是演示代码

```
int i;
	struct hostent *host;
	
	host=gethostbyname("www.baidu.com");
	if(!host)
		error_handling("gethost... error");

	printf("Official name: %s \n", host->h_name);
	
	for(i=0; host->h_aliases[i]; i++)
		printf("Aliases %d: %s \n", i+1, host->h_aliases[i]);
	
	printf("Address type: %s \n", 
		(host->h_addrtype==AF_INET)?"AF_INET":"AF_INET6");

	for(i=0; host->h_addr_list[i]; i++)
		printf("IP addr %d: %s \n", i+1,
					inet_ntoa(*(struct in_addr*)host->h_addr_list[i]));
	return 0;
```
另外根据ip也可以获得域名 `struct hostent * gethostbyaddr(const char * addr, socklen_t len, int family);`演示代码

```
	struct hostent *host;
	struct sockaddr_in addr;

	memset(&addr, 0, sizeof(addr));
	addr.sin_addr.s_addr=inet_addr(argv[1]);
	host=gethostbyaddr((char*)&addr.sin_addr, 4, AF_INET);

```
也就是说一个域名可以对应多个ip，一个ip也可以有多个域名。一个域名对应多个ip显然至少可以做负载均衡用。一个ip对应多个域名，可以更好的推广。



## 结束