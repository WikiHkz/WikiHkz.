---
layout: post
title: Linux的IO复用与epoll
date: 2023-10-23
categories: master
tags: recommend
keywords: Linux
description: 
---

## 基础概念

### 文件描述符fd

文件描述符（File descriptor）是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念。

文件描述符在形式上是一个**非负整(unsingned int)数**。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。

### I/O模式

对于一次IO访问（以read举例），数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。所以说，当一个read操作发生时，它会经历两个阶段：

1. 等待数据准备 (Waiting for the data to be ready)
2. 将数据从内核拷贝到进程中 (Copying the data from the kernel to the process)

正式因为这两个阶段，linux系统产生了下面五种网络模式的方案。

- 阻塞 I/O（blocking IO）
- 非阻塞 I/O（nonblocking IO）
- I/O 多路复用（ IO multiplexing）
- 信号驱动 I/O（ signal driven IO）
- 异步 I/O（asynchronous IO）

#### 阻塞 I/O（blocking IO）

在linux中，默认情况下所有的socket都是blocking，一个典型的读操作流程大概是这样：

当用户进程调用了recvfrom这个系统调用，kernel就开始了IO的第一个阶段：准备数据（对于网络IO来说，很多时候数据在一开始还没有到达。比如，还没有收到一个完整的UDP包。这个时候kernel就要等待足够的数据到来）。这个过程需要等待，也就是说数据被拷贝到操作系统内核的缓冲区中是需要一个过程的。而在用户进程这边，整个进程会被阻塞（当然，是进程自己选择的阻塞）。当kernel一直等到数据准备好了，它就会将数据从kernel中拷贝到用户内存，然后kernel返回结果，用户进程才解除block的状态，重新运行起来。

> 所以，blocking IO的特点就是在IO执行的两个阶段都被block了。

#### 非阻塞 I/O（nonblocking IO）

linux下，可以通过设置socket使其变为non-blocking。当对一个non-blocking socket执行读操作时，流程是这个样子：

当用户进程发出read操作时，如果kernel中的数据还没有准备好，那么它并不会block用户进程，而是立刻返回一个error。从用户进程角度讲 ，它发起一个read操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦kernel中的数据准备好了，并且又再次收到了用户进程的system call，那么它马上就将数据拷贝到了用户内存，然后返回。

> 所以，nonblocking IO的特点是用户进程需要**不断的主动询问**kernel数据好了没有。

#### I/O 多路复用（ IO multiplexing）

IO multiplexing就是我们说的select，poll，epoll，有些地方也称这种IO方式为event driven IO。select/epoll的好处就在于单个process就可以同时处理多个网络连接的IO。它的基本原理就是select，poll，epoll这个function会不断的轮询所负责的所有socket，当某个socket有数据到达了，就通知用户进程。

`当用户进程调用了select，那么整个进程会被block`，而同时，kernel会“监视”所有select负责的socket，当任何一个socket中的数据准备好了，select就会返回。这个时候用户进程再调用read操作，将数据从kernel拷贝到用户进程。

> 所以，I/O 多路复用的特点是通过一种机制一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，select()函数就可以返回。

如果处理的连接数不是很高的话，使用select/epoll的web server不一定比使用multi-threading + blocking IO的web server性能更好，可能延迟还更大。select/epoll的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。）

在IO multiplexing Model中，实际中，对于每一个socket，一般都设置成为non-blocking，但是，整个用户的process其实是一直被block的。只不过process是被select这个函数block，而不是被socket IO给block。

#### 异步 I/O（asynchronous IO）

用户进程发起read操作之后，立刻就可以开始去做其它的事。而另一方面，从kernel的角度，当它受到一个asynchronous read之后，首先它会立刻返回，所以不会对用户进程产生任何block。然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel会给用户进程发送一个signal，告诉它read操作完成了。

## I/O复用

select，poll，epoll都是IO多路复用的机制。I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

### select

```
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述副就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以 通过遍历fdset，来找到就绪的描述符。

#### 原理

```c
	sockfd = socket(AF_INET, SOCK_STREAM, 0);
  memset(&addr, 0, sizeof (addr));
  addr.sin_family = AF_INET;
  addr.sin_port = htons(2000);
  addr.sin_addr.s_addr = INADDR_ANY;
  bind(sockfd,(struct sockaddr*)&addr ,sizeof(addr));
  listen (sockfd, 5); 

  for (i=0;i<5;i++) 
  {
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    fds[i] = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    if(fds[i] > max)
    	max = fds[i];
  }
  
  while(1){
	FD_ZERO(&rset);
  	for (i = 0; i< 5; i++ ) {
  		FD_SET(fds[i],&rset);
  	}

   	puts("round again");
	select(max+1, &rset, NULL, NULL, NULL);

	for(i=0;i<5;i++) {
		if (FD_ISSET(fds[i], &rset)){
			memset(buffer,0,MAXBUF);
			read(fds[i], buffer, MAXBUF);
			puts(buffer);
		}
	}	
  }
  return 0;
}
```

select的原理比较简单粗暴，Linux中使用文件描述符集合`  fd_set rset;`,这是一个bitmap，在Linux的宏中被定义为1024个，select会监听这个文件描述符集合。

1. 每一个新建的文件描述符，都会分配一个数值，这个数值是唯一的，不可重复的，并且一定在bitmap中的。
2. 创建文件描述符集合`  fd_set rset;`
3. 把需要监听的文件描述符，在文件描述符集合bitmap中置位，需要监听的文件描述符置为1，select会监听bitmap中置为1的文件描述符
4. select调用时，会把bitmap值为1的文件描述符从用户态**拷贝**到内核态并进行监听
5. select调用后进行阻塞，如果有被监听的文件描述符有数据进来，select会把对应的bitmap置位，没有数据的bitmap置0，然后返回
6. 返回后，遍历检查bitmap为1的文件描述符，读取数据并进行处理
7. **重新在bitmap中置位**需要监听的文件描述符，并使用select进行监听

总结下来：

调用select时，**总是会破坏**保存需要监听的文件描述符的bitmap，因为select同样使用bitmap返回了当前有数据需要处理的文件描述符，因此再每次调用select返回之后，都要重新设置bitmap。

需要注意的还有：

1. 当被监听的文件描述符有数据时，select就会置位相应的bitmap然后返回，有多个数据同时到达时，会同时置位所有有数据到达的文件描述符
2. select会返回有数据到达的文件描述符的数量
3. 无法通过select的返回值直接去取数据，必须要遍历bitmap为1的文件描述符，因为文件描述符在bitmao中并不是顺序存储的

select的缺点：

1. bitmap的大小默认是1024，这对可监听的文件描述符数量有了限制，但是可以通过修改宏定义进行修改
2. FDset不可重用，每次都需要重新设置
3. 从用户态到内核态到切换与拷贝需要有较大的开销
4. select返回后仍然要有一个On的遍历才可以处理数据

### poll

```c
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

不同与select使用bitmap来表示三个fdset的方式，poll使用一个 pollfd的指针实现。

```c
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};	
```

#### 原理

```c
for (i=0;i<5;i++) 
  {
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    pollfds[i].fd = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    pollfds[i].events = POLLIN;
  }
  sleep(1);
  while(1){
  	puts("round again");
	poll(pollfds, 5, 50000);

	for(i=0;i<5;i++) {
		if (pollfds[i].revents & POLLIN){
			pollfds[i].revents = 0;
			memset(buffer,0,MAXBUF);
			read(pollfds[i].fd, buffer, MAXBUF);
			puts(buffer);
		}
	}
  }
```

poll的原理与select不同，其主要的实现方式是通过一个结构体`struct pollfd`。

select监听文件描述的行为是通过不同的文件描述符集合来实现的，writefds、readfds、和exceptfds，而poll是通过struct pollfd中的events来描述。如果有多个功能需要监听，直接多个功能或即可。

```c
pollfds[i].events = POLLIN | POLLERR; // 监听输入和错误
```

在poll中，不在使用文件描述集，而是使用一个`struct pollfd`数组。每一个文件描述符都会被`struct pollfd`修饰后存放在pollfd数组中`pollfds[]`。

在poll函数的参数中，传入这个数组的地址和长度即可。

- struct pollfd

这个结构体中，只有三个变量，分别表示了当前的文件描述符`int fd;`，当前文件描述符需要监听的功能`short events; `，以及这个文件描述符发生的事情描述`short revents;`。

poll也是一个阻塞函数，知道所监听的文件描述符产生对应的事件，有数据需要处理时才会返回，也是返回发生事件的文件描述符的数量。

与select不同的是：

1. 针对不同功能监听，select使用了三个文件描述符集bitmap；而poll通过使用结构体，每个文件描述符的功能监听都写在各自的pollfd结构体中
2. 发生了对应的事件后，select是把对应的bitmap置位；而poll是把对应的文件描述符的pollfd结构体的revents进行置位，通过检查revents来判断该文件描述符的events是否发生

poll的原理可以总结为：

1. 为每一个文件描述符使用pollfd结构体进行修饰，并使用数组存放
2. poll函数在调用时使用pollfd结构体数组作为参数传递
3. poll函数在调用时会把所有需要监听的文件描述符从用户态**拷贝**到内核态中进行监控，并进行阻塞
4. 如果对应文件描述符的对应监听事件发生了，内核会把该文件描述符的pollfd结构体中revents进行置位，并返回发生事件的文件描述符的数量
5. 返回后通过**遍历**pollfd结构体数组，检查其中revents的状态，如果revents状态不为0，则表示对应的events事件发生
6. events事件发生后，需要**手动把revents置0！！！**
7. 进行数据处理
8. 重新调用poll进行监听

缺点：

1. poll解决了select中bitmap默认为1024的问题，理论上pollfd结构体数组的长度为无限制
2. poll解决了select中FDset不能重用的问题，只需要处理数据前把events修改即可，pollfd结构体数组可以重复使用
3. poll仍然没有解决调用时从用户态到内核态切换与文件描述符拷贝的开销
4. poll返回后，仍然需要遍历整个pollfd结构体数组来检查其中revents的状态来判断事件是否发生

### epoll

epoll是在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

1. 红黑树（Red-Black Tree）：`epoll` 使用红黑树来管理所有已注册的文件描述符。红黑树是一种自平衡二叉搜索树，可以提供快速的查找操作，使内核能够有效地查找需要通知的文件描述符。
2. 双向链表（Doubly-Linked List）：`epoll` 使用双向链表来管理已触发事件的文件描述符，即那些已经准备好进行 I/O 操作的文件描述符。这些文件描述符在链表上链接，以便内核可以有效地跟踪它们。

#### epoll操作过程

epoll操作过程需要三个接口，分别如下：

```c
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

**1. int epoll_create(int size);**
创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大，这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值，参数size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。
当创建好epoll句柄后，它就会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

**2. int epoll_ctl(int epfd, int op, int fd, struct epoll_event \*event)；**
函数是对指定描述符fd执行op操作。
- epfd：是epoll_create()的返回值。

- op：表示op操作，用三个宏来表示：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD。分别添加、删除和修改对fd的监听事件。

- fd：是需要监听的fd（文件描述符）

- epoll_event：是告诉内核需要监听什么事，struct epoll_event结构如下：

  ```c
  struct epoll_event {
    __uint32_t events;  /* Epoll events */
    epoll_data_t data;  /* User data variable */
  };
  
  //events可以是以下几个宏的集合：
  EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
  EPOLLOUT：表示对应的文件描述符可以写；
  EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
  EPOLLERR：表示对应的文件描述符发生错误；
  EPOLLHUP：表示对应的文件描述符被挂断；
  EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
  EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
  ```

  **3. int epoll_wait(int epfd, struct epoll_event \* events, int maxevents, int timeout);**
  等待epfd上的io事件，最多返回maxevents个事件。
  参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。

  ``epoll_wait` 的调用将会阻塞，直到有一个或多个文件描述符上发生了事件，或者超时发生。当它返回时，用户可以访问事件数组中的信息，以确定哪些文件描述符需要处理。这种机制允许用户高效地等待和处理多个事件，而不需要手动轮询文件描述符。
  
  直接通过events指针与函数的返回值，即可直接O1的处理所有有事件触发的文件描述符。

#### 工作模式

epoll对文件描述符的操作有两种模式：**LT（level trigger）**和**ET（edge trigger）**。LT模式是默认模式，LT模式与ET模式的区别如下：
**LT模式**：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，`应用程序可以不立即处理该事件`。下次调用epoll_wait时，会再次响应应用程序并通知此事件。
**ET模式**：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，`应用程序必须立即处理该事件`。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

##### 1. LT模式

LT(level triggered)是缺省的工作方式，并且同时支持block和no-block socket.在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的。

##### 2. ET模式

ET(edge-triggered)是高速工作方式，只支持no-block socket。在这种模式下，如果监听的文件描述符发生了对应的事件，则称为文件描述符是就绪状态。内核会通过eopll告诉用户该文件描述处于就绪状态，并且不再为其发送更多的通知(例如当前接收到数据后，用户接收到就绪状态，读取数据，但是没有读取完毕，但即使这样，再次调用epoll_wait时也不会再重新发布就绪通知)。

ET 模式会通知在文件描述符上发生状态变化的时刻。这意味着只有在文件描述符从无数据变为有数据、从不可写变为可写等状态变化时，epoll 才会通知事件。

这意味着，一旦 `epoll_wait` 返回某个socket上发生了事件，用户程序必须确保在下一次调用 `epoll_wait` 之前处理了该事件。如果事件未被处理，`epoll` 将不会再次通知它，即使socket上的状态仍然是可以读取数据的。

ET模式要求对应使用的socket是非阻塞的，原因在于，如果如果使用阻塞套接口，当某个事件发生时，程序会在处理该事件之前一直阻塞在 `read` 或 `write` 调用上，直到数据可用或操作完成。如果事件已经通知过一次（因为 ET 模式只通知一次），并且您未立即读取所有可用数据或处理错误，那么将无法再次获得事件通知，从而可能导致事件被遗漏。

#### epoll总结

在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而**epoll事先通过epoll_ctl()来注册一 个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知**。(`此处去掉了遍历文件描述符，而是通过监听回调的的机制`。这正是epoll的魅力所在。)

epoll还解决了用户态和内核态之间文件描述符拷贝开销的问题。一旦文件描述符注册到 `epoll` 实例后，内核会维护这些文件描述符的事件状态，而不需要用户程序再次拷贝文件描述符集合。当事件发生时，内核只通知用户程序已经发生事件的文件描述符，而不需要频繁的拷贝整个文件描述符集合。
