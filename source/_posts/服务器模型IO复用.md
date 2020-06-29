---
title: IO复用技术
abbrlink: 250739322
date: 2020-05-15 10:24:30
tags:
---
在之前的项目中，服务器可以接收客户端连接，并通过子线程将其写入到数据库中。这次我们将利用IO复用技术提高系统性能。项目地址 [github](https://github.com/mequanwei/wServer)
<!--more-->

## 阻塞和异步
在之前的项目中，我们采用 accept() 来接收新的连接。accept() 方法是一种阻塞的方法，也就意味着，主线程会一直在这里等待，直到收到一个新的来连接。这样子阻塞的效率是很低下的。通过 setsockopt 我们可以将 socket 设置成非阻塞式，这样线程就不会阻塞在accept，这样需要每隔一段时间取读取 
socket 状态。

有一种机制，能够在 accept 不阻塞的同时，当 accept 收到新的连接时便调用回调自动处理。这便是异步，与之对应的是同步，需要主动查询accept状态并作出的响应。boost 中的 asio 类提供了一种异步机制。

## IO 复用
linux开发中常用一种技术是IO复用。前面提到，对同步 I/O 而言，阻塞I/O有一个比较明显的缺点是在I/O阻塞模式下，一个线程只能处理一个流的I/O事件。非阻塞的I/O需要轮询查看流是否已经准备好了，比较典型的方式是忙轮询。I/O 复用指得是同时监听多个请求，以提高系统利用率。
常用的 IO复用有 select, poll 和 epoll
### selsect
 select监听一组进程描述符集合，程序会阻塞在select函数上，直到被监视的文件句柄中有一个或多个发生了状态变化。
 
 ```
 int select(int maxfd,fd_set *readset,fd_set *writeset,fd_set *exceptset,const struct timeval *timeout)
 ```
maxfd 参数为监听的文件描述符最大值+1, readset,writeset,exceptset 是三个需要监听的集合，将需要监听可读的描述符放到readset，将需要监听可写的描述符放到 writeset，将需要监听异常状态的描述符放到 exceptset, 然后设置超时时间，即可开始监听。
select 的返回值表示监听集合中状态改变的个数，需要手动查询集合，以获取状态变化的描述符。

select的最大缺点就是进程打开的fd是有数量限制的。默认最大支持 1024。selsect 机制采用的是线性查找机制，即内核会依次查找从 0 到 maxfd 的文件描述符，当需要监听的集合很大时，花费的时间将呈线性增长。同时 select 需要将监听集合先拷贝到内核空间，修改后，再拷贝到用户空间。因此 select 的效率并不高。

### poll
Poll的处理机制与Select类似:
```
int poll ( struct pollfd * fds, unsigned int nfds, int timeout);
```
poll的缺点同select。

### epoll
epoll是在 linux 2.6 中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。 epoll 有三个相关接口。
```c++
int epoll_create(int size)；
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```
首先需要通过 epoll_create 创建一个句柄，size用来告诉内核这个监听的数目一共有多大。如要监听 1000 个连接，就设置为 1000。然后通过 epoll_ctl 设置监听的集合，epfd 即 epoll 句柄，op为集合操作，如 EPOLL_CTL_ADD 表示添加，fd 和 event 分别是需要监听的描述符和监听的事件。最后，通过 epoll_wait() 等待返回结果。项目重构后，epoll 使用的实例如下：
```c++
int Server::start()
{
	int ret;
	int server_socket;
	int status;
    int number;
	char msg[BUFFER_SIZE]; /*buffer*/

	struct sockaddr_in ser_addr; /* server address */
	struct sockaddr_in cli_addr; /* client address */

	server_socket = socket(AF_INET, SOCK_STREAM, 0);
	if (server_socket < 0)
	{
		printf("open socket error\n");
		return 0;
	}

	ser_addr.sin_family = AF_INET;
	ser_addr.sin_addr.s_addr = htonl(ACCEPTED_ADDRESS);
	ser_addr.sin_port = htons(PORT);

	ret = bind(server_socket, (struct sockaddr*) &ser_addr,
			sizeof(struct sockaddr_in));
	if (ret < 0)
	{
		printf("bind error\n");
		return 0;
	}

	ret = listen(server_socket, BACKLOG);
	if (ret < 0)
	{
		printf("listening error\n");
		return 0;
	}

	epollfd = epoll_create(EPOLL_SIZE);	
	addfd(epollfd, server_socket);

	while (1)
	{
		number = epoll_wait(epollfd, events, EPOLL_SIZE, -1);
		for (int i = 0; i < number; i++)
		{
			if (events[i].data.fd == server_socket)
			{
				int client_socket = accept(server_socket, NULL, NULL);  //第二三个参数用记录连接的客户端状态
				if (client_socket <0)
				{
					cout <<"error"<<endl;
					continue;
				}
				addfd(epollfd, client_socket);
			}
			else if (events[i].events & EPOLLIN)
			{
				handleRead(events[i].data.fd);
			}
			else if (events[i].events & EPOLLOUT)
			{
				handleWrite(events[i].data.fd);
			}
		}
	}

	return 0;
}


void Server::addfd(int epollfd, int fd) {
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN |EPOLLET |EPOLLRDHUP|EPOLLOUT;

    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);

    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;

    fcntl(fd, F_SETFL, new_option);
}
```

### epoll 的工作模式
epoll对两种工作模式：LT（level trigger）和ET（edge trigger）。LT模式是默认模式，LT模式与ET模式的区别如下：
LT模式下，当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。LT(level triggered)是缺省的工作方式。

ET(edge-triggered)是高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知。

在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描。
而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait()时便得到通知。此处去掉了遍历文件描述符，而是通过监听回调的的机制。
epoll 监视的描述符数量不受限制，它所支持的FD上限是最大可以打开文件的数目。而且，epoll的 IO的效率不会随着监视fd的数量的增长而下降。epoll不同于select和poll轮询的方式，是通过每个fd定义的回调函数来实现的。只有就绪的fd才会执行回调函数。


## 总结
重构之后，我们的服务器通过一个主线程监听请求，当收到一个请求之后，epoll_wait() 监听到，如果有变化的描述符是我们创建的socekt，表示有新的连接到来，如果不是，则判断是返回状态是可读还是可写，通过将其加入线程池等待队列，对描述符做出对应的处理。

