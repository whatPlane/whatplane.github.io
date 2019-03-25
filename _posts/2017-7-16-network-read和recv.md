---
layout:     post                    # 使用的布局（不需要改）
title:      read/write/recv/send              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 网络
---

## 区别
read和write是一对，recv和send是一对。他们的作用是一样的。主要的区别是recv/send多了第四个参数。

```
ssize_t read(int fd,void *buf,size_t nbyte)
ssize_t write(int fd, const void*buf,size_t nbytes);

int recv(int sockfd,void *buf,int len,int flags)
int send(int sockfd,void *buf,int len,int flags)

```

## 最后一个变量是解释
| MSG_DONTROUTE | 不查找表 |
| MSG_OOB | 接受或者发送带外数据 |
| MSG_PEEK |MSG_DONOTWAIT | 这两个可以联合使用查看数据,并不从系统缓冲区移走数据，不阻塞。类似stack的top。
| MSG_WAITALL | 等待所有数据 |

- **MSG_DONTROUTE**:是send函数使用的标志.这个标志告诉IP.目的主机在本地网络上面,没有必要查找表.这个标志一般用网络诊断和路由程序里面.
- **MSG_OOB**:后面具体演示怎么使用

- **MSG_PEEK**:是recv函数的使用标志,表示只是从系统缓冲区中读取内容,而不清除系统缓冲区的内容.这样下次读的时候,仍然是一样的内容.一般在有多个进程读写数据时可以使用这个标志.

- **MSG_WAITALL** 是recv函数的使用标志,表示等到所有的信息到达时才返回.使用这个标志的时候recv回一直阻塞,直到指定的条件满足,或者是发生了错误. 
  - 1)当读到了指定的字节时,函数正常返回.返回值等于len 
  - 2)当读到了文件的结尾时,函数正常返回.返回值小于len 
  - 3)当操作发生错误时,返回-1,且设置错误为相应的错误号(errno)


## 带外数据
这个主要用于tcp的紧急通知。tcp的片段里面可以设置紧急位和紧急指针。特别的是，这个紧急数据只能一次只能发送一个字节。接受数据在特别是缓冲中提取出来。使用的场景是：服务器处理繁忙时，也就是没有及时调用read函数读取数据，这时客户端发送紧急信息来催促服务器要加油处理。服务器一般注册一个信号处理函数，在信号处理函数里面接受紧急数据。因为服务器收到紧急数据后，会在当前进程中设置一个紧急位的信号，触发信号响应函数。紧急数据并不会影响到正常数据的顺序。注意需要设置处理sigurg 信号的进程。

```
	fcntl(recv_sock, F_SETOWN, getpid()); 

```
代码的意思是，recv_sock引发的 **sigurg** 信号由getpid（）所在的进程来处理。如果有多个进程，他们因为父子关系，所以可能有相同的信号处理函数，那么信号产生的时候，该通知哪个进程来处理呢？这就是这句代码的含义。下面是服务器截取代码：

```
void urg_handler(int signo)
{
	
	str_len=recv(recv_sock, buf, sizeof(buf)-1, MSG_OOB);
	
}
```
下面是客户端发送代码：

```
send(sock, "4", strlen("4"), MSG_OOB);
```


## 结束