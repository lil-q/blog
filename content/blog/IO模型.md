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

大多数文件系统的默认 I/O 操作都是缓存 I/O，缓存 I/O 又被称作标准 I/O，。在 Linux 的缓存 I/O 机制中，操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

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

nonblocking IO 的特点是用户进程需要**不断的主动询问** kernel 中的数据是否就绪，每次轮询所有 FD（包括没有发生读写事件的 FD）会很浪费 CPU 资源。

### 1.3 I/O 多路复用

2003年，[C10K 问题](https://link.zhihu.com/?target=http%3A//www.kegel.com/c10k.html)迅速成为了技术领域迫切需要尝试解决的问题：10K 个 Client 同时连接到一个 Server，如何提供服务？：在互联网用户较少的早期，一个网络服务通常不会有太多用户，加之大多数的服务不过是提供一个 HTML，交互较少，所以往往使用的是一个进程服务一个连接的形式；而进程不仅仅存在进程数限制，还存在进程切换和通讯带来的巨大代价。改用一个线程服务一个连接模型，能够一定程度减少部分开销，但也只是勉强够用而已。更好的解决办法在哪里？

为了解决同步阻塞 I/O 面临的一个链路需要一个线程处理的问题，后来有人对它的线程模型进行了优化——后端通过一个线程池来处理多个客户端的请求接入，形成客户端个数 M：线程池最大线程数 N 的比例关系，其中 M 可以远远大于 N。这是一种伪异步 I/O 通信框架，采用线程池实现，因此避免了为每个请求都创建一个独立线程造成的线程资源耗尽问题。不过因为它的底层仍然是同步阻塞的 BIO 模型，因此无法从根本上解决问题。

但是如果改成 NIO，那么每个线程都需要不断的主动轮询，这是对 CPU 资源的极大浪费。再重新回想 I/O 的过程，第一部分等待数据就绪的过程和 CPU 关系不大，更加取决于机器缓存大小和网络带宽等，而第二部分拷贝数据需求 CPU 性能。如果让一个额外的线程统一管理其他线程等待数据就绪的过程，也就是说让一个线程完成所有其他线程的主动轮询工作，一次轮询就可以处理多个连接请求，这就是I/O 多路复用（ IO multiplexing）。所以，**多路**指的是多个网络连接，**复用**指的是复用同一个线程。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after9.9/nio3.png" alt="nonblockingio" style="zoom:150%;" />

I/O 多路复用的特点是通过一种机制让一个进程能同时等待多个文件描述符，而这些文件描述符（套接字描述符）其中的任意一个进入读就绪状态，`select()` 函数就可以返回。

需要注意的是，使用 I/O 多路复用技术时，线程在调用 `select()` 时虽然没有被阻塞，但是  `select()` 在所需求的任意一个 FD 状态发生改变后才返回，相当于被阻塞了，与 blocking IO 相比甚至还多了一个 system call (recvfrom)。如果处理的连接数不是很高的话，使用 I/O 多路复用的 web server 不一定比使用 multi-threading + blocking IO 的 web server 性能更好，可能延迟还更大。I/O 多路复用的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接。

### 1.4 异步 I/O

上述三种 I/O 模型都属于同步 I/O，因为不管是哪一种，拷贝数据的过程都是阻塞的。

> \- A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes;<br>\- An asynchronous I/O operation does not cause the requesting process to be blocked.

用户进程发起 read 操作之后，立刻就可以开始去做其它的事。而另一方面，从 kernel 的角度，当它受到一个 asynchronous read 之后，首先它会立刻返回，所以不会对用户进程产生任何 block。然后，kernel 会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel 会给用户进程发送一个 signal，告诉它 read 操作完成了。

<img src="https://qttblog.oss-cn-hangzhou.aliyuncs.com/after9.9/nio4.png" alt="nonblockingio" style="zoom:150%;" />

异步 I/O（asynchronous IO）的特点是线程无需自己负责读写，异步 I/O 的实现负责把数据从内核拷贝到用户空间。

## 二、SELECT, POLL & EPOLL



## 参考

1. [Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)
2. UNIX® Network Programming Volume 1, Third Edition: The Sockets Networking
3. [也来说说协程（1）——由来和概念](https://zhuanlan.zhihu.com/p/204965836)
4. [BIO,NIO,AIO 总结](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/BIO-NIO-AIO.md)
5. [彻底理解 IO多路复用](https://juejin.im/post/6844904200141438984)