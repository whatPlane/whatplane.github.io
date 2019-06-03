---
layout:     post                    # 使用的布局（不需要改）
title:      read/write/recv/send              # 标题 
subtitle:     #副标题
date:       2017-7-17              # 时间
author:     co                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - network
---
## read/write 
产生的sock都有对应的内核接收缓冲区和发送缓冲区。大小是os自动调节。write的本质是把应用层的数据写入到发送缓冲区，至于什么时候发送，全靠os。read的本质是取出接收缓冲区的数据。read/write在阻塞和非阻塞模式下有不同的表现。sock的关闭也可以通过read/write的error反应出来。对端关闭主要分为两种情况，一种是对端的进程异常关闭，这种情况因为os还在，所以fin会依然发送出来，另一种情况是对端主机宕机或者网络断开，这种情况是无法发送出fin的。


- 阻塞模式下

  - 当write的时候，比如说打算写入100字节，但是发送缓冲区只有50字节空闲了，那么此时wirte就会阻塞，直到发送缓冲区数据被发送出去，可以容纳剩下的50字节，才会返回。read操作是只要接收缓冲区有数据，那么就会返回，函数的返回值代表了实际读取的数据大小。如果接受缓冲区没有数据，此时调用read，就会阻塞，直到接收缓冲区有收到对端发过来的数据。

  - 如果write阻塞的时候，对端的进程异常关闭（不管进程是否正常发送fin片段，os都会在进程退出的时候，关闭文件描述符，这样导致一定会发送fin片段）发送了fin，write端会直接返回，返回值是已经写入发送缓冲区的数据大小。再次调用write会失败，connection reset by peer。
  - 如果是read阻塞，对端进程异常关闭，当然会返回0。因为接收缓冲区收到了fin片段。此时再进行write操作，当然不会有问题，毕竟收到对端的fin只表示对端关闭了写缓冲，还是可以接收数据的。在刚刚的write发送数据后，对端会回应一个rst，说明对端已经重置了。当然这个rst我们已经收到，但是应用层还不知道。当再次write的时候，就会引发sigpipe错误，默认是终止进程。当然我们一般会忽略这个信号。
  - 如果我们read正阻塞，对端的主机奔溃或者出现网络断开问题，那么read会永远阻塞下去。
  - 如果我们write正在阻塞，对端的主机奔溃或者出现网络断开问题，那么由于发送缓冲区的数据无法得到对端的确认ack，tcp重发的机制会不断的尝试重发。重发时间和重发次数超过一次阈值时，write会返回。通知设置error为超时或者主机无法到达（ETIMEDOUT/EHOSTUNREACH/ENETUNREACH）。如果在这重发期间，对端的主机恢复了，那么我们会收到rst，表示对端已经重置了。


> write 阻塞的原因是发送缓冲区满了 ，read阻塞的原因是接收缓冲区没有数据可读。
> 一般来说通讯时发送速度总是比接收速度快。

- 非阻塞模下
  - write的时候，如果发送缓冲区没有空闲，则直接返回-1，设置error为EAGIN。如果发送缓冲区空闲空间不够，也会立即返回，并且函数返回值是实际复制到发送缓冲区的数据大小。
  - read的时候，都会立即返回。返回值如果是-1，error为EAGIN，则表示接收缓冲区没有数据可读，如果返回值大于0则是实际读取的数据大小，为0则表示对端已经关闭sock。
  - 如果对方进程异常关闭，我们调用write可以成功，接着会收到对方发送的rst，再次调用write，write会返回-1，error为sigpipe。
  - 如果对方主机奔溃或者网络断开，我们可以一直read，但是读不到返回值为0，无法判断对方是否关闭。但是通过write同样会跟在阻塞模式下一样，通过tcp重发最终发现对方已经出现问题。**也就是说这种情况只有通过write才能检查出问题**。
  - 下面是设置sock为非阻塞模式的方法

```
// 设置一个文件描述符为nonblock
int set_nonblocking(int fd)
{
    int flags;
    if ((flags = fcntl(fd, F_GETFL, 0)) == -1)
        flags = 0;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}
```




## read/write 区别 recv/send
read和write是一对，recv和send是一对。他们的作用是一样的。主要的区别是recv/send多了第四个参数。

```
ssize_t read(int fd,void *buf,size_t nbyte)
ssize_t write(int fd, const void*buf,size_t nbytes);

int recv(int sockfd,void *buf,int len,int flags)
int send(int sockfd,void *buf,int len,int flags)

```

## flags的可选项解释
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