---
title: TCP连接的优雅关闭
abbrlink: 3098900096
date: 2020-07-12 09:26:54
tags:
---

所谓优雅关闭是指，如果发送缓存中还有数据未发出则其发出去，并且收到所有数据的ACK之后，发送FIN包，然后关闭socket。
<!--more-->

## 关闭的时机
对服务器而言，有三种情况需要关闭socket。
1. 当连接太久没有活动，定时器超时。
2. 当客户端关闭，发送FIN时。此时 read/write 操作会返回0。并且 errno=EAGAIN
3. 当客户端异常关闭，没有发送FIN，此时 read/write 会返回异常。

## 怎样关闭
1. close
close(fd) 调用会将描述字的引用计数减1，只有当socket描述符的引用计数为0时，才关闭socket，即发送FIN包，因此，在fork()时，父进程在accept()返回后，fork()子进程，由子进程处理connfd，而父进程将close(connfd);由于connfd这个socket描述符的引用计数不为0，因此并不引发FIN，所以就没有关闭和客户端的连接。  主动端发送fin后，这个套接字就被不能再用 write 和 read 了，在应用进程层面就是销毁的意义了，剩下的工作内核会自动完成。

2. shutdown(fd，how) 
shutdown() 函数允许只停止在某个方向上的数据传输，而一个方向上的数据传输继续进行。如你可以关闭某socket的写操作而允许继续在该socket上接受数据，直至读入所有数据。
其中fd是需要关闭的socket的描述符。参数 how允许为shutdown操作选择以下几种方式：
    - SHUT_RD：关闭连接的读端。也就是该socket不再接受数据，任何当前在接受缓冲区的数据将被丢弃。进程将不能对该socket发出任何读操作。对socekt该调用之后接受到的任何数据将被确认然后无声的丢弃掉。
    - SHUT_WR:关闭连接的写端，进程不能在对此socket发出写操作。
    - SHUT_RDWR:相当于调用shutdown两次：首先是以SHUT_RD,然后以SHUT_WR。
使用close中止一个连接，但它只是减少描述符的参考数，并不直接关闭连接，只有当描述符的参考数为0时才关闭连接。shutdown可直接关闭描述符，不考虑描述符的参考数，可选择中止一个方向的连接。但是 shutdown 与 socket 描述符没有关系，即使调用shutdown(fd, SHUT_RDWR)也不会关闭fd，仅仅关闭了TCP连接，最终还需关闭文件描述符close(fd)。

3. SO_LINGER 选项
```c++
struct linger ling;
ling.l_onoff = 1;
ling.l_linger = 0;
setsockopt(fd, SOL_SOCKET, SO_LINGER, (char*)&ling, sizeof(ling));
close(fd);
```
将 SO_LINGER 设置为开启并且 l_linger 非零值时，close的行为取决于两个条件：一是被关闭的socket对应的TCP发送缓冲区是否还有残留的数据；二是该socket是阻塞的，还是非阻塞的。对于阻塞的socket，close将等待一段长为l_linger的时间，直到TCP模块发送完所有残留数据并得到对方的确认。如果这段时间内TCP模块没有发送完残留数据并得到对方的确认，那么close系统调用将返回-1并设置errno为EWOULDBLOCK。如果socket是非阻塞的，close将立即返回，此时我们需要根据其返回值和errno来判断残留数据是否已经发送完毕。
将 SO_LINGER 设置为开启并且 l_linger 为零时，socket 将丢弃保留在发送缓冲区中的任何数据并发送一个RST给对方，而不是通常的四次挥手，这避免了TIME_WAIT状态。
