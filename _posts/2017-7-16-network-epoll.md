---
layout:     post                    # 使用的布局（不需要改）
title:      epoll模型              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 网络
---
## epoll模型跟select模型的主要区别
epoll的内部实现是红黑树+链表。树主要用来管理监听的sock，每个sock都在网卡驱动程序里注册了回调函数。一旦sock身上有事件发生，则把对应的sock加入到链表。
epoll主要改进了select的两个问题。
- select每次调用都要重新传入监听集合。也就是说如果监听了7个描述符，如果下次还是继续监听这个7个描述符，需要再次传入监听集合给select。这是一个很大的开销
- select为了找出有事件发生的描述符，需要遍历所有的监听描述符，再通过FD_ISSET(fd, &cpy_reads)来确定。

这两点正式epoll改进的了地方。epoll设置监听通过epoll_ctl，返回的集合就是发生事件的集合，不需要像select那样查找出来。使用epoll模型需要关注的主要有三个函数和一个结构体 struct epoll_event
- epoll_create 
- epoll_wait
- epoll_ctl


```
struct epoll_event {

    __uint32_t events;      // Epoll events

    epoll_data_t data;      // User datavariable

};

typedef union epoll_data {

    void *ptr;

    int fd;

    __uint32_t u32;

    __uint64_t u64;

} epoll_data_t;

```
## 函数介绍

```
intepoll_create(int size);

```
size表示提供给系统的参考值，表示你期望监听的sock的最大个数。

```

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

```

- epfd就是之前产生的epoll。op如下三个选项
  - EPOLL_CTL_ADD  
  - EPOLL_CTL_DEL
  - EPOLL_CTL_MOD 
- fd是监听的sock的描述符
- event是监听事件。其中event.events有下面几个主要的选项
  - EPOLLIN  可读
  - EPOLLOUT 可写
  - EPOLLET 边缘触发模式
  - EPOLLPRI OOB带外数据
  - EPOLLONESHOT 发生一次事件后，相应sock不再收到通知。只有再次调用epoll_ctl（epfd，EPOLL_CTL_MOD，fd，event） 后才可以。这个后面会讲。


```
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```
- epfd 即epoll_create 产生的fd
- events 堆上分配的数组
- maxevents 数组的大小
- timeout 是超时时间 单位是毫秒也就是 1/1000 秒,传递-1时表示一直阻塞，直到有事件发生




epoll模型基本使用流程
- 通过epoll_create 产生一个epoll。
- 在堆空间分配产生一个数组，里面含有一定数量的 epoll_event 对象。作为参数传递给epoll_wait用。
- 通过epoll_ctl来控制监听的fd。
- 最后调用epoll_wait来获取发生的事件

## 一个演示代码
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>

#define BUF_SIZE 100
#define EPOLL_SIZE 50
void error_handling(char *buf);

int main(int argc, char *argv[])
{
	int serv_sock, clnt_sock;
	struct sockaddr_in serv_adr, clnt_adr;
	socklen_t adr_sz;
	int str_len, i;
	char buf[BUF_SIZE];

	struct epoll_event *ep_events;
	struct epoll_event event;
	int epfd, event_cnt;

	if(argc!=2) {
		printf("Usage : %s <port>\n", argv[0]);
		exit(1);
	}

	serv_sock=socket(PF_INET, SOCK_STREAM, 0);
	memset(&serv_adr, 0, sizeof(serv_adr));
	serv_adr.sin_family=AF_INET;
	serv_adr.sin_addr.s_addr=htonl(INADDR_ANY);
	serv_adr.sin_port=htons(atoi(argv[1]));
	
	if(bind(serv_sock, (struct sockaddr*) &serv_adr, sizeof(serv_adr))==-1)
		error_handling("bind() error");
	if(listen(serv_sock, 5)==-1)
		error_handling("listen() error");

	epfd=epoll_create(EPOLL_SIZE);
	ep_events=malloc(sizeof(struct epoll_event)*EPOLL_SIZE);

	event.events=EPOLLIN;
	event.data.fd=serv_sock;	
	epoll_ctl(epfd, EPOLL_CTL_ADD, serv_sock, &event);//event不需要额外分配空间

	while(1)
	{
		event_cnt=epoll_wait(epfd, ep_events, EPOLL_SIZE, -1);
		if(event_cnt==-1)
		{
			puts("epoll_wait() error");
			break;
		}

		for(i=0; i<event_cnt; i++)
		{
			if(ep_events[i].data.fd==serv_sock)
			{
				adr_sz=sizeof(clnt_adr);
				clnt_sock=
					accept(serv_sock, (struct sockaddr*)&clnt_adr, &adr_sz);
				event.events=EPOLLIN;
				event.data.fd=clnt_sock;
				epoll_ctl(epfd, EPOLL_CTL_ADD, clnt_sock, &event);//event不需要额外分配空间
				printf("connected client: %d \n", clnt_sock);
			}
			else
			{
					str_len=read(ep_events[i].data.fd, buf, BUF_SIZE);
					if(str_len==0)    // close request!
					{
						epoll_ctl(
							epfd, EPOLL_CTL_DEL, ep_events[i].data.fd, NULL);
						close(ep_events[i].data.fd);
						printf("closed client: %d \n", ep_events[i].data.fd);
					}
					else
					{
						write(ep_events[i].data.fd, buf, str_len);    // echo!
					}
	
			}
		}
	}
	close(serv_sock);
	close(epfd);
	return 0;
}

``` 

## ET和LT 触发模式

***
|     事件      |  缓冲区状态                      | 电平 |
| :-----------------: | :---------------------: | :---------------------: |
| EPOLLIN |  不为空  | 高电平|
| EPOLLIN |  空  | 低电平|
| EPOLLOUT |  不满  | 高电平|
| EPOLLOUT |  满  | 低电平|

> 讨论接收缓冲区的时候，我们关注空，因为空是非阻塞read函数返回的条件，同理，讨论发送缓冲区的时候，关注是否满，满是非阻塞write返回的条件。


EPOLLIN
- 如果sock接收缓冲区 不为空 认为是 高电平
- 如果sock接收缓冲区 空    认为是 低电平

EPOLLOUT
- 如果sock发送缓冲区 不满  认为是 高电平
- 如果sock发送缓冲区 满    认为是 低电平

LT触发的条件是 **高电平**
ET出发点条件是 高--》低 或者是 低--》高

***

LT触发是sock默认的选项。假设接收缓冲收到了100字节数据，如果应用程序收到可读事件通知，并读取了50字节。
- 如果是LT触发模式，则下次调用epoll_wait时，还会收到可读事件通知。这样可以把剩下的50字节读取出来。
- 如果是ET触发模式，则下次调用epoll_wait时，不会收到可读事件通知。所以一般在ET模式下，收到可读事件通知时都会循环调用read，直到read返回-1，error设置为EAGIN。也就是把缓冲区的数据完全读取出来。**如果此时sock接收缓冲又收到数据，epoll_wait会再次触发可读事件通知吗**？答案是不会。ET模式触发的条件是，从无到有。因为此时接收缓冲区还有数据，再接收数据，还是处于有数据的状态，没有发现状态跃迁。

> ET模式必须工作在非阻塞模式。如果不工作在非阻塞模式，那么当有可读事件发生时，想通过循环完全读取缓冲区的数据将导致阻塞，其他sock的通知事件就无法处理了。因为你无法判断数据是否已经完全读完了。而LT模式却可以工作在阻塞模式。因为我们默认LT模式下，只调用一次read，这样收到可读事件时，读取固定长度数据总是会返回，不用担心被阻塞了。


> ET模式在accept新连接的时候，会同时关注读写事件，这个跟LT此时只关注读事件不同。而且ET在写的时候，需要完全填充内核发送缓冲区。也就是write的时候，要看到返回 EAGIN为止，不然后面无法再次收到可写事件。


## EPOLLONESHOT 控制
假设如下场景：一个sock监听了可读事件，如果一个sock的接收缓冲区收到数据，epoll通知`线程1`去读取sock的数据，如果此时epoll发现sock又收到数据，此时再通知`线程2`对sock进行读操作，显然导致两个线程对同一个sock进行操作。设置EPOLLONESHOT选项可以保证事件只通知一次，只有后续再次进行epoll_ctl才会收到通知。

## 什么时候write数据
一般我们发送数据的流程是这样。直接调用write。然后根据实际复制到发送缓冲区的数据大小，来决定是否需要关注sock的可写事件。也就是通过epoll_ctl来监听EPOLLOUT事件。如果需要监听可读事件，那么当发送缓冲区的数据被os取走后，epoll得到可写事件时，再把之前剩余的数据写进缓冲区。



## 结束