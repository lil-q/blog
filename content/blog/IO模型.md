---
title: "I/O 模型"
date: 2020-10-07T20:30:35+08:00
slug: ""
description: "介绍了阻塞 I/O、非阻塞 I/O、I/O 多路复用、异步 I/O，SELECT，POLL 和 EPOLL"
keywords: [I/O, BIO, NIO, I/O多路复用]
tags: [web]
math: false
toc: true
---

**I/O**（**I**nput / **O**utput），即**输入／输出**，通常指数据在存储器（内部和外部）或其他周边设备之间的输入和输出，是信息处理系统与外部世界之间的通信。输入是系统接收的信号或数据，输出则是从其发送的信号或数据。

大多数文件系统的默认 I/O 操作都是缓存 I/O，缓存 I/O 又被称作标准 I/O。在 Linux 的缓存 I/O 机制中，操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

数据在传输过程中需要在应用程序地址空间和内核进行多次数据拷贝操作，这些数据拷贝操作所带来的 CPU 以及内存开销是非常大的。当一个 read 操作发生时，会经历两个阶段：

1. 等待数据准备（Waiting for the data to be ready）；
2. 将数据从内核拷贝到进程中（Copying the data from the kernel to the process）。

对于一个套接字上的输入操作，第一步通常涉及等待数据从网络中到达。当所等待分组到达时，它被复制到内核中的某个缓冲区。第二步就是把数据从内核缓冲区复制到应用进程缓冲区。

## 一、I/O 模型

linux 系统下有五种网络模式的方案，其中信号驱动 I/O 使用较少，本文将介绍其余四种模型。

* 阻塞 I/O（blocking IO）
* 非阻塞 I/O（nonblocking IO）
* I/O 多路复用（ IO multiplexing）
* 信号驱动 I/O（ signal driven IO）
* 异步 I/O（asynchronous IO）

在开始介绍之前，需要对文件描述符（File Descriptor）有一定认识：

> 文件描述符是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念。文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于 UNIX、Linux 这样的操作系统。

### 1.1 阻塞 I/O

在 linux 中，默认情况下所有的 socket 都是阻塞 I/O（blocking IO），一个典型的读操作流程大概是这样：

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after9.9/nio1.png" alt="blockingio" style="zoom:150%;" />

阻塞 I/O 的特点是等待数据和拷贝数据两个过程都是阻塞的。

### 1.2 非阻塞 I/O

linux 下，可以通过设置 socket 使其变为非阻塞 I/O（non-blocking I/O）。当对一个 non-blocking socket 执行读操作时，流程是这个样子：

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after9.9/nio2.png" alt="nonblockingio" style="zoom:150%;" />

当用户进程发出 read 操作时，如果 kernel 中的数据还没有准备好，会立刻返回一个 error。从用户进程角度讲 ，它发起一个 read 操作后，并不需要等待，而是马上就得到了一个结果。用户进程判断结果是一个 error 时，它就知道数据还没有准备好，于是它可以再次发送 read 操作。一旦 kernel 中的数据准备好了，并且又再次收到了用户进程的 system call，那么它马上就将数据拷贝到了用户内存，然后返回。

nonblocking IO 的特点是用户进程需要**不断的主动询问** kernel 中的数据是否就绪，每次轮询所有 FD（包括没有发生读写事件的 FD）会很浪费 CPU 资源。需要注意的是，线程在阻塞过程中是不占用 CPU 的。

### 1.3 I/O 多路复用

2003年，[C10K 问题](https://link.zhihu.com/?target=http%3A//www.kegel.com/c10k.html)迅速成为了技术领域迫切需要尝试解决的问题：10K 个 Client 同时连接到一个 Server，如何提供服务？在互联网用户较少的早期，一个网络服务通常不会有太多用户，加之大多数的服务不过是提供一个 HTML，交互较少，所以往往使用的是一个进程服务一个连接的形式；而进程不仅仅存在进程数限制，还存在进程切换和通讯带来的巨大代价。改用一个线程服务一个连接模型，能够一定程度减少部分开销，但也只是勉强够用而已。更好的解决办法在哪里？

为了解决同步阻塞 I/O 面临的一个链路需要一个线程处理的问题，后来有人对它的线程模型进行了优化——后端通过一个线程池来处理多个客户端的请求接入，形成客户端个数 M：线程池最大线程数 N 的比例关系，其中 M 可以远远大于 N。这是一种伪异步 I/O 通信框架，采用线程池实现，因此避免了为每个请求都创建一个独立线程造成的线程资源耗尽问题。不过因为它的底层仍然是同步阻塞的 BIO 模型，因此无法从根本上解决问题。

但是如果改成 NIO，那么每个线程都需要不断的主动轮询，这是对 CPU 资源的极大浪费。再重新回想 I/O 的过程，第一部分等待数据就绪的过程和 CPU 关系不大，更加取决于机器缓存大小和网络带宽等，而第二部分拷贝数据需求 CPU 性能。如果让一个额外的线程统一管理其他线程等待数据就绪的过程，也就是说让一个线程完成所有其他线程的主动轮询工作，一次轮询就可以处理多个连接请求，这就是I/O 多路复用（ IO multiplexing）。所以，**多路**指的是多个网络连接，**复用**指的是复用同一个线程。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after9.9/nio3.png" alt="nonblockingio" style="zoom:150%;" />

I/O 多路复用的特点是通过一种机制让一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，`select()` 函数就可以返回。

需要注意的是，使用 I/O 多路复用技术时，线程在调用 `select()` 时也是被阻塞的，因为 `select()` 在所需求的任意一个 FD 状态发生改变后才返回，与 blocking IO 相比甚至还多了一个 system call (recvfrom)。如果处理的连接数不是很高的话，使用 I/O 多路复用的 web server 不一定比使用 multi-threading + blocking IO 的 web server 性能更好，可能延迟还更大。I/O 多路复用的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。

### 1.4 异步 I/O

上述三种 I/O 模型都属于同步 I/O，因为不管是哪一种，拷贝数据的过程都是阻塞的。

> \- A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes;<br>\- An asynchronous I/O operation does not cause the requesting process to be blocked.

用户进程发起 read 操作之后，立刻就可以开始去做其它的事。而另一方面，从 kernel 的角度，当它受到一个 asynchronous read 之后，首先它会立刻返回，所以不会对用户进程产生任何 block。然后，kernel 会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel 会给用户进程发送一个 signal，告诉它 read 操作完成了。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after9.9/nio4.png" alt="nonblockingio" style="zoom:150%;" />

异步 I/O（asynchronous IO）的特点是线程无需自己负责读写，异步 I/O 的实现负责把数据从内核拷贝到用户空间。

## 二、I/O 多路复用中的函数

Java 中的 NIO 就是采用的多路复用机制，在不同的操作系统有不同的实现，在 unix 和 linux 上是 `select()`，`poll()` & `epoll()`。`select()` 在 32 位系统下，单进程最多支持打开 1024 个文件描述符；`poll()` 对其进行了一些优化，能打开的文件描述符不受限制（但还是要取决于系统资源）；而 `epoll()` 的出现是为了解决上述两种方法都存在的性能问题。

### 2.1 SELECT

该函数允许进程指示内核等待多个事件中的任何一个发生，并只在有一个或多个事件发生或经历一段指定的时间后才唤醒它。
作为一个例子，我们可以调用 `select()`，告知内核仅在下列情况发生时才返回：

* 集合 {1,4,5} 中的任何描述符准备好读；
* 集合 {2,7} 中的任何描述符准备好写；
* 集合 {1,4} 中的任何描述符有异常条件待处理；
* 已经历了 10.2 秒。

也就是说，我们调用 `select()` 告知内核对哪些描述符（就读、写或异常条件）感兴趣以及等待多长时间。我们感兴趣的描述符不局限于套接字，任何描述符都可以使用 `select()` 来测试。

```c
int select (int maxfdp1, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

***maxfdp1*** 参数指定待测试的描述符个数，它的值是待测试的最大描述符加 1（max_fd_plus_1），描述符0, 1, 2 … 一直到 maxfdp1 - 1 均将被测试。

***readfds***，***writefds***，***exceptfds*** 是三个位图，指定我们要让内核测试读、写和异常条件的描述符。目前支持的异常条件只有两个：

* 某个套接字的带外数据的到达，我们将在第24章中详细讲述这个异常条件；
* 某个已置为分组模式的伪终端存在可从其主端读取的控制状态信息。

***timeout*** 表示超时时长，超过该时长就需要返回，timeval 结构用于指定这段时间的秒数和微秒数。这个参数有以下三种可能：

* 永远等待下去：仅在有一个描述符准备好 IO 时才返回，该参数设置为空指针；
* 等待一段固定时间：在有一个描述符准备好 IO 时返回，时长不超过由该参数所指向的 timeval 结构中指定的秒数和微秒数；
* 根本不等待：检查描述符后立即返回，这称为轮询（polling），该参数指向的 timeval 结构中指定的秒数和微秒数为 0。

该函数的**返回值**表示跨所有描述符集的已就绪的总位数；如果在任何描述符就绪之前定时器到时，那么返回 0；如果出错（如本函数被一个所捕获的信号中断）返回 -1。

#### 1. 位图的限制

我们以一个 16 位的位图为例，简述 `select()` 的过程，假设我们需要查询 0 号、3 号和 7 号 FD 是否可读，就需要传入如下 *readfds* 位图：

```text
1001000100000000
```

当 3 号 FD 准备就绪时，会保留已就绪 FD 的位并把其他未就绪的 FD 的位置 0 后返回：

```text
0001000000000000
```

应用进程收到返回信号，当时他并不知道哪个 FD 已就绪，所以应用进程会**遍历位图**来找到所以就绪的 FD。

通过上述过程不难发现，`select()` 存在以下缺陷：

* 每一次查询后都需要遍历位图，这是一笔相当大的开销；
* 由于使用位图来记录 FD，所以存在最大文件描述符数量限制，在 Linux 上一般为 1024。

### 2.2 POLL

`poll()` 提供的功能与 `select()` 类似，不过在处理流设备时，它能够提供额外的信息。

```c
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

不同与 `select()` 使用三个位图来表示三个 *fdset* 的方式，`poll()` 使用 *pollfd* 结构数组来实现。

```c
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```

*pollfd* 结构包含了要监视的 *event* 和发生的 *event*，不再使用 `select()` 中 “参数-值” 传递的方式。同时，*pollfd* 并**没有最大数量限制**（但是数量过大后性能也是会下降）。 和 `select()` 函数一样，`poll()` 返回后，需要轮询 *pollfd* 数组来获取就绪的描述符。

`poll()` 和 `select()` 相同都需要在返回后，通过**遍历文件描述符**来获取已经就绪的 *socket*。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。

### 2.3 EPOLL

`epoll()` 在 2.6 内核中提出，是之前 `select()` 和 `poll()` 的增强版本。`epoll()` 操作过程需要三个接口，如下：

```c
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

`epoll()`  相较于 `select()` 多了一个 *eventpoll*，其中持有一个包含已就绪 FD 的等待队列 *rdlist* 以及包含当前阻塞线程的等待队列，之前需要监视的 *socket* 都绑定一个进程用来唤醒，现在都改为指向 *eventpoll* ，具体使用时的伪代码如下：

```java
// 假设现目前获得了很多 serverSocket.accept()
sockets = getSockets(); 
// 创建 eventpoll
int epfd = epoll_create();
// 将所有需要监视的 socket 都加入到 eventpoll 中
epoll_ctl(epfd, sockets);
while (true) {
    // 阻塞返回准备好了的 sockets
    int n = epoll_wait();
    // 直接对收到数据的 socket 进行遍历，不需要再遍历所有的 sockets
    // 是怎么做到的呢，下面继续分析
    for (遍历接收到数据的 socket) {
        
    }
}
```

这里的等待队列表示 *eventpoll* 上挂起的进程，是处于被阻塞状态的从工作队列移除的，需要被唤醒。

就绪队列 *rdlist* 是 *eventpoll* 的一个成员，指的是内核中有哪些数据已经准备就绪。当调用 `epoll_ctl()` 的时候会为每一个 *socket* 注册一个回调函数，当某个 *socket* 准备好了就会**回调**然后加入 *rdlist* 中的，*rdlist* 的数据结构是一个双向链表。

`epoll()` 提升了系统的并发能力，较于 `select()`、`poll()` 具有以下优势：

- 内核监视 *socket* 的时候不再需要每次传入所有的 *socket* 文件描述符，然后又全部断开（唤醒进程后是需要解绑的）的操作了，只需通过一次 epoll_ctl 即可；
- `select()`、`poll()` 模型下进程收到了 *socket* 准备就绪的指令执行后，它不知道到底是哪个 *socket* 就绪了，需要去遍历所有的 *socket*，而 `epoll()` 维护了一个 *rdlist* 通过回调的方式将就绪的 *socket* 插入到 *rdlist* 链表中，我们可以直接获取 *rdlist* 即可，无需遍历其它的 *socket*，提升效率。

#### 1. EPOLL 工作模式

`epoll()` 对文件描述符的操作有两种模式：**LT**（**L**evel **T**rigger）和 **ET**（**E**dge **T**rigger）。LT 模式是默认模式，LT 模式与 ET 模式的区别如下：

* **LT模式**：当 `epoll_wait()` 返回后，应用程序**可以不立即处理该事件**。如果不处理，下次调用 `epoll_wait()` 时，会再次响应应用程序并通知此事件。
* **ET模式**：当 `epoll_wait()` 返回后，应用程序**必须立即处理该事件**。如果不处理，下次调用`epoll_wait()` 时，不会再次响应应用程序并通知此事件。

## 参考

1. [Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)
2. UNIX® Network Programming Volume 1, Third Edition: The Sockets Networking
3. [也来说说协程（1）——由来和概念](https://zhuanlan.zhihu.com/p/204965836)
4. [BIO,NIO,AIO 总结](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/BIO-NIO-AIO.md)
5. [彻底理解 IO多路复用](https://juejin.im/post/6844904200141438984)
6. [多路复用 I/O 模型详解, 为什么它能支持更高的并发](https://juejin.im/post/6844903954917097486#heading-5)

