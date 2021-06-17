---
title: "Java 并发：BlockingQueue"
date: 2021-06-17T13:14:27+08:00
slug: ""
description: ""
keywords: [java, concurrency]
tags: [java, concurrency]
math: false
toc: true
---

*BlockingQueue* 阻塞队列支持多个线程同时对其入队和出队。

<br>

| 方法                                    | 操作成功  |         操作失败          |
| --------------------------------------- | --------- | :-----------------------: |
| add(E e)                                | 返回 true |         抛出异常          |
| put(E e)                                | void      |         一直阻塞          |
| offer(E e)                              | 返回 true |        返回 false         |
| offer(E e, long timeout, TimeUnit unit) | 返回 true | 等待 timeout 后返回 false |

生产者-消费者模式的本质就是用**缓存区**将生产和消费**解耦**，提高并发效率。

