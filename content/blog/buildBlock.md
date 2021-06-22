---
title: "Java 并发：BlockingQueue"
date: 2021-06-17T13:14:27+08:00
slug: ""
description: ""
keywords: [java, concurrency]
tags: [java, concurrency]
draft: true
math: false
toc: true
---

*BlockingQueue* 阻塞队列接口是支持多个线程并发处理的先入后出队列。类似于 *List*，它有两个常用的实现类 *ArrayBlockingQueue* 和 *LinkedBlockingQueue*。

```java
// 容量无上限
BlockingQueue<String> blockingQueue = new ArrayBlockingQueue<>(); 
// 容量上限10
BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>(10); 
```

将元素 e 入队有以下方法：

<br>

| 方法                                    | 操作成功  |         操作失败          |
| --------------------------------------- | --------- | :-----------------------: |
| put(E e)                                | void      |         一直阻塞          |
| add(E e)                                | 返回 true |         抛出异常          |
| offer(E e)                              | 返回 true |        返回 false         |
| offer(E e, long timeout, TimeUnit unit) | 返回 true | 等待 timeout 后返回 false |

`put(E e)` 属于阻塞调用，直到入队成功才会返回；与之相对的，调用 `add(E e)` 和 `offer(E e)` 时，不管入队成功与否都会立刻返回，而不会等待。这里的等待有两种可能：当前有其他线程正在处理队列，需要等待争用；或者队列已满，需要等待其他线程出队。还有一个比较折中的办法就是调用 `offer(E e, long timeout, TimeUnit unit)`，它会等待 timeout 的时间，还是无法入队就返回 false，其中 unit 是指定的时间单位。

出队获得元素 e 有以下方法：

<br>

| 方法                              | 操作成功 | 操作失败                 |
| --------------------------------- | -------- | ------------------------ |
| take()                            | 返回 e   | 一直阻塞                 |
| poll(long timeout, TimeUnit unit) | 返回 e   | 等待 timeout 后返回 null |

`take()` 属于阻塞调用，直到出队成功才会返回，如果其他线程正在处理队列或者队列为空，将一直阻塞。

## 一、生产者-消费者模式

*BlockingQueue* 常见的应用场景就是生产者-消费者模式。



### 1.1 广义的线程限制



生产者-消费者模式中使用 *BlockingQueue* 的本质就是利用**缓存区**将生产和消费**解耦**，提高并发效率。



## 参考



