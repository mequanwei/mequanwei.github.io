---
title: Linux TCP服务器——通过socket建立连接
abbrlink: 3924114082
date: 2020-03-27 05:03:42
---

疫情期间将丢了好久的 C++ 后台开发相关知识捡了回来，同时萌生了写一个小项目的想法。可以将已有知识融会贯通，以后找工作还能展示一下。先从最简单的 socket 通信开始，随后有什么想法，新技术之类的都会加在项目中。在此记录项目的完善过程，项目源码在 [github](https://github.com/mequanwei/wServer)。
<!--more-->

## socket 
客户端和服务器依靠 socket 通信。socket是一种特殊的文件，socket 相关函数就是对其进行读写，打开关闭等操作。在网络编程中，很多应用层协议都是对socket的封装。当创建一个 socket 时，操作系统就返回一个文件描述符来标识这个套接字。然后，应用程序以该描述符作为传递参数，通过调用函数来完成操作。

## 连接过程
整体通信过程如下图。
![img](../img/socket-01.jpg)

## 具体过程和API
socket 调用需要用到 glibc 提供的接口，这些接口都在 sys/socket.h 中。主要用的的几个接口如下。
首先介绍服务端需要做的工作。
```c++
int socket(int domain, int type, int protocol);
```
socket 函数返回一个文件描述符用于通信。其第一个参数是协议域，一般取 AF_INET 表示IPV4 协议。第二个参数表示协议类型，常用取值有 SOCK_STREAM 代表 TCP、SOCK_DGRAM代表UDP。第三个参数表示指定协议，一般取0表示自动选择。

在打开了 socket 之后，socket不知道具体的通信地址。如果想要给它赋值一个地址，就必须调用 bind() 函数。
``` c++
int bind(int socket, const struct sockaddr *address, socklen_t address_len);
sockaddr_in ser_addr;
ser_addr.sin_family = AF_INTR;
ser_addr.s_aaddr = htonl (ACCEPTED_ADDRESS)
ser_addr.sin_port = htons(PORT)
```
bind 函数将 socket 和一个通信地址绑定。这个地址的类型和之前选择的协议域有关，如果是IPV4协议，那么对应的类型是 sockaddr_in。通过 IP 和端口号标识位置。bind() 函数的第一个参数是之前生成的 socket 描述符。第二个参数是地址结构体，第三个参数是地质结构体的长度。因为这里要对地址结构体进行强制类型转换，所以需要长度信息。

绑定了地址信息之后需要将 socket 转为监听状态。socket 默认是主动连接模式，只能连接其他socket，通过调用listen()函数可以将其转为可以接收其他socket 消息的装态。
```c++
int listen(int socket, int backlog);
```
第一个参数是 socket 描述符，第二个参数表示最大等待的连接个数。当超过这个连接个数时，系统将丢弃新来的来连接请求。

在设置了监听状态之后，便可以通过 accept 开始监听。
```c++
int accept(int socket, struct sockaddr *restrict address, socklen_t *restrict address_len);
```
第一个参数是 socket 描述符，第二个参数是一个地址结构体，用于接收连接到socket上的客户端信息。第三个参数时地址结构体的长度。

客户端的过程和服务端类似，需要创建 socket 和绑定地址信息。接下来如果时 tcp 连接，需要调用 connect 函数进行三次握手。如果是 udp 连接，则不需要此步骤。
```c++
int connect(int socket, const struct sockaddr *address,socklen_t address_len);
```
第一个参数是 socket 描述符，第二个参数是一个地址结构体，用于接收服务端socket的信息。第三个参数时地址结构体的长度。

连接建立之后，就可以通过 send/recv,sendto/recvfrom 来通信。这两组函数功能一致，但是 sendto 可以在没有建立连接的情况使用，在使用时需要传入地址信息。
```c++
ssize_t send (int __fd, const void *__buf, size_t __n, int __flags);
ssize_t sendto (int __fd, const void *__buf, size_t __n, int __flags, __CONST_SOCKADDR_ARG __addr, socklen_t __addr_len);
```

最后，通信完成记得关闭socket
```c++
int close(int fildes);
```

## 实例代码
这里展示了一个 tcp 服务器，该服务器接收到客户端信息之后，调用一个子进程将该信息写回给客户端。然后继续监听其他客户端请求。
### 客户端

```c++
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <netinet/in.h>

#define ACCEPTED_ADDRESS INADDR_ANY
#define PORT 1234
#define BACKLOG 2
#define BUFFER_SIZE 1024

int main()
{
    int server_socket;
    int client_socket;
    char msg[BUFFER_SIZE]; /*buffer*/
    struct sockaddr_in ser_addr; /* server address */
    struct sockaddr_in cli_addr; /* client address,unused */    
    //create socket
    //int socket(int domain, int type, int protocol);
    server_socket = socket(AF_INET,SOCK_STREAM,0);
    if (server_socket<0){
    	printf("open socket error");
    	return 0;
    }
    //bind socket
    ser_addr.sin_family=AF_INET;
    ser_addr.sin_addr.s_addr=htonl(ACCEPTED_ADDRESS);
    ser_addr.sin_port=htons(PORT);    
    if(bind(server_socket,(struct sockaddr*)&ser_addr,sizeof(struct sockaddr_in)) < 0){
    	printf("bind error");
    	return 0;
    }    
    //set listening status;
    if(listen(server_socket,BACKLOG)<0){
    	printf("listening error");
    	return 0;
    }
    printf("begin listening");
    
    //	handle listening
    socklen_t len = sizeof(struct sockaddr);
    while(1){
    //	client_socket = accept(server_socket,(struct sockaddr*)&cli_addr,(unsigned int *)sizeof(struct sockaddr_in));
    	client_socket = accept(server_socket,(struct sockaddr*)&cli_addr,&len);
    	if (client_socket<0){
    		printf("handle error");
    		return 0;
        }
        printf("server start working..");
        recv(client_socket, msg, (size_t)BUFFER_SIZE, 0);
        printf("received a connection from %s\n", inet_ntoa(cli_addr.sin_addr));
        printf("%s\n",msg);
        strcpy(msg,"hello,client,this is server");
        send(client_socket, msg, sizeof(msg),0); //sent msg
        close(client_socket);
    }
    close(server_socket);
}
```
### 客户端
```c++
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <netinet/in.h>
#include <errno.h>

#define SERVER_ADDRESS 0X7F000001
#define BUFFER_SIZE 1024
#define PORT 1234
int main()
{
    int server_socket;
    int client_socket;
    char msg[BUFFER_SIZE]; /*buffer*/
    //struct sockaddr_in ser_addr; /* server address */
    struct sockaddr_in cli_addr; /* client address,unused */
    client_socket = socket(AF_INET,SOCK_STREAM,0);
    if (client_socket<0){
    	printf("open socket error");
    	return 0;
    }
    cli_addr.sin_family=AF_INET;
    cli_addr.sin_addr.s_addr=inet_addr("127.0.0.1");
    cli_addr.sin_port= htons(PORT);
    int err_code= connect(client_socket,(struct sockaddr *)&cli_addr,sizeof(sockaddr_in));
    if(err_code <0){
    	printf("connect error:%d",err_code);
    	return 0;
    }
    int len=0;
    //begin working
    printf("success connect to %u\n",SERVER_ADDRESS);
    	strcpy(msg,"hello server,this is client");
    	send(client_socket,msg,strlen(msg),0);
    	recv(client_socket, msg, (size_t)BUFFER_SIZE, 0);
    	printf("%s",msg);
    close(client_socket);
}
```

## 一些问题
1. udp 协议的socket也能调用 connect，不过不会进行三次握手，内核会记录下服务器的地址信息。一般用于打开新的 udp 连接和断开连接。
2. 服务器listen之后，客户端便可以 connect 进行三次握手，可以通过 tcpdump抓到相关包。建立的连接会放到一个队列中等待 accept 使用。
3. accept 函数会从已经建立连接的队列中取出第一个连接，并创建一个新的socket，新的socket的类型和地址参数要和原来的那个指定的socket的地址一一样，并且还要为这个新的socket分配文件描述符。新建的这个socket自身是无法再接收连接了，但是最开始的socket仍然是处于开放状态，而且可以接收更多连接。
4. send 函数只是将要发送的内容写入了内核缓存区，对端不一定接收到了。
5. 如果一直不调用recv函数，内核接收缓存区会满，这时 tcp 协议通过滑动窗口通知对端不要在发送，而 udp 协议则会直接丢弃。
