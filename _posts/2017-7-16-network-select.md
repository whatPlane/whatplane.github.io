---
layout:     post                    # 使用的布局（不需要改）
title:      select模型              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - network
---

## select
函数介绍
```
struct timeval｛

long tv_sec;//秒数

long tv_usec;//微秒数

｝;

int select(int nfds, fd_set *readfds, fd_set *writefds,fd_set *exceptfds,
 			struct timeval *timeout);
```
- nfds 表示监听的描述符集合中最大值再加1， 即 maxfd + 1
- 有事件发生，则返回值表示已经准备好的描述符个数。
- readfds writefds exceptfds 是分别需要监听的三个集合。设置为null则表示不关注。
- timeout是超时设置。
  - 设为为null 则表示一直阻塞等待，知道有事件发生。
  - 如果 `timeout->tv_sec=0 && timeout->tv_usec=0` 则表示设置select成非阻塞
  - 如果 `timeout->tv_sec！=0 || timeout->tv_usec!=0`则设置超时等待，如果到超时事件还没有事件发生则返回0
  



## fd_set
fd_set可以看做是一个位集合。可以把对应的位设置为1和0.这个位跟描述符是对应的。观察下图
![](https://gitee.com/whatplane/resource/raw/master/img/wx_20190326090505-min.png)

## select模型流程
使用select模型的流程和注意点
- 每次调用select，需要重新设置监听fd_set。这个集合是select函数的输入参数也是输出参数。我们经常采会copy一份fd_set来保存之前的修改。具体看后面的代码。
- select的第一个参数是 maxfd+1 ，maxfd是监听集合中的最大描述符。
- 当发生事件后，fd_set把发生事件的位设置为1，其他位设为0
- 应用程序通过遍历找出发生事件的fd。一般我们会通过一个数组把fd都保存起来，然后通过调用FD_ISSET(fd,fd_set)来检测事件是否发生。
- 如果监听sock上有读事件发生，那么说明有新连接产生。
- 如果select返回值是-1，说明有错误。如果error = EINTR ，则表示被信号终端。
- 添加到fd_set中的fd值必须小于FD_SETSIZE。



## 一段示例代码
注意这段代码演示中，描述符并没有通过一个数组保存起来。

```
	int serv_sock, clnt_sock;
	struct sockaddr_in serv_adr, clnt_adr;
	struct timeval timeout;
	fd_set reads, cpy_reads;

	socklen_t adr_sz;
	int fd_max, str_len, fd_num, i;
	char buf[BUF_SIZE];
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

	FD_ZERO(&reads);
	FD_SET(serv_sock, &reads);
	fd_max=serv_sock;

	while(1)
	{
		cpy_reads=reads; 这行代码很重要
		timeout.tv_sec=5;
		timeout.tv_usec=5000;

		if((fd_num=select(fd_max+1, &cpy_reads, 0, 0, &timeout))==-1)
			break;
		
		if(fd_num==0)
			continue;

		for(i=0; i<fd_max+1; i++)
		{
			if(FD_ISSET(i, &cpy_reads))
			{
				if(i==serv_sock)     // connection request!
				{
					adr_sz=sizeof(clnt_adr);
					clnt_sock=
						accept(serv_sock, (struct sockaddr*)&clnt_adr, &adr_sz);
					FD_SET(clnt_sock, &reads);更新监听集合
					if(fd_max<clnt_sock)
						fd_max=clnt_sock;
					printf("connected client: %d \n", clnt_sock);
				}
				else    // read message!
				{
					str_len=read(i, buf, BUF_SIZE);
					if(str_len==0)    // close request!
					{
						FD_CLR(i, &reads);从监听集合中取消一个监听
						close(i);
						printf("closed client: %d \n", i);
					}
					else
					{
						write(i, buf, str_len);    // echo!
					}
				}
			}
		}
	}
	close(serv_sock);
	return 0;
```
## select作为定时器
主要是利用超时功能。select(0,NULL,NULL,NULL,&tv); 这里演示一个秒级定时器
```
void seconds_sleep(unsigned seconds){
    struct timeval tv;
    tv.tv_sec=seconds;
    tv.tv_usec=0;
    int err;
    do{
       err=select(0,NULL,NULL,NULL,&tv);
    }while(err<0 && errno==EINTR);
}
```
## 结束