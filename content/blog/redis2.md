---
title: "Redis：过期键淘汰策略"
date: 2020-10-14T20:36:54+08:00
slug: ""
description: "介绍Redis几种过期键淘汰机制"
keywords: [Redis, 淘汰]
tags: [Redis]
math: false
toc: true
---

## 一、过期键的判定

*redisDb* 结构的 *expires* 字典保存了数据库中所有键的过期时间，我们称这个字典为过期字典：

* 过期字典的键是一个指针，这个指针指向键空间中的某个键对象（即数据库键）；
* 过期字典的值是一个 *longlong* 类型的整数，这个整数保存了键所指向的数据库键的过期时间个毫秒精度的 *UNIX* 时间戳。

通过过期字典，程序可以用以下步骤检查一个给定键是否过期：

1. 检査给定键是否存在于过期字典：如果存在，那么取得键的过期时间；
2. 检查当前 *UNIX* 时间戳是否大于键的过期时间：如果是的话，那么键已经过期；否则的话，键未过期。

## 二、过期键淘汰策略

常见的过期键淘汰策略有以下三种：

* 定时删除：在设置键的过期时间的同时，创建一个定时器（timer），让定时器在键的过期时间来临时，立即执行对键的删除操作。
* 惰性删除：放任键过期不管，但是每次从键空间中获取键时，都检查取得的键是否过期，如果过期的话，就删除该键；如果没有过期，就返回该键。
* 定期删除：每隔一段时间，程序就对数据库进行一次检査，删除里面的过期键。至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。

### 2.1 定时删除

* **优点：对内存最友好**，使用定时器可以保证过期键尽快被删除，并释放过期键所占内存。

* **缺点：对 CPU 时间最不友好**，在过期键比较多的情况下，删除过期键这一行为可能会占用相当一部分 CPU 时间，会对服务器的响应时间和吞吐量造成影响。

除此之外，创建一个定时器需要用到 Redis 服务器中的时间事件，而当前时间事件的实现方式——无序链表，查找一个事件的时间复杂度为 O(M) 并不能高效地处理大量时间事件。

因此，要让服务器创建大量的定时器，从而实现定时删除策略，在现阶段来说并不现实。

### 2.2 惰性删除

* **优点：对 CPU 时间最友好**，程序只会在取出键时才对键进行过期检査，不会在删除其他无关的过期键上花费任何 CPU 时间。
* **缺点：对内存最不友好**，如果一个键已经过期，而这个键又仍然保留在数据库中，那么只要这个过期键不被删除，它所占用的内存就不会释放。

在使用惰性删除策略时，如果数据库中有非常多的过期键，而这些过期键又正好没有被访问到的话，那么它们也许永远也不会被删除（除非用户手动执行 FLUSHDB），这可以看作是一种内存泄漏——无用的垃圾数据占用了大量的内存。

### 2.3 定期删除

由上述可知，定时删除和惰性删除实际上是两个极端。定时删除占用太多 CPU 时间，影响服务器的响应时间和吞吐量；惰性删除浪费太多内存，有内存泄漏的危险。

定期删除策略是前两种策略的一种整合和折中。定期删除策略每隔一段时间执行一次删除过期键操作，并通过限制删除操作执行的时长和频率来减少删除操作对 CPU 时间的影响。除此之外，通过定期删除过期键，定期删除策略有效地减少了因为过期键而带来的内存浪费。按照如下规则确定删除操作执行的时长和频率：

* 如果删除操作执行得太频繁，或者执行的时间太长，定期删除策略就会退化成定时删除策略，以至于将 CPU 时间过多地消耗在删除过期键上面。
* 如果删除操作执行得太少，或者执行的时间太短，定期删除策略又会和惰性删除策略一样，出现浪费内存的情况。

因此，如果采用定期删除策略的话，服务器必须根据情况，合理地设置删除操作的执行时长和执行频率。

## 三、Redis 过期键淘汰策略

Redis 服务器实际使用的是惰性删除和定期删除两种策略，执行时 Redis 有以下具体淘汰策略：

* *noeviction*（默认）：当内存使用超过配置的时候会返回错误，不会删除任何键；
* *allkeys-lru*：从所有键中通过 LRU 算法删除最久没有使用的键；
* *volatile-lru*：从设置过期时间的数据集中删除最久没有使用的键；
* *allkeys-random*：从所有 key 键中随机删除；
* *volatile-random*：从设置过期时间的数据集中随机删除；
* *volatile-ttl*：从设置过期时间的数据集中删除马上就要过期的键，*ttl* 值越大越优先被淘汰；
* *volatile-lfu*：从设置过期时间的数据集中删除使用频率最少的键；
* *allkeys-lfu*：从所有键中驱逐使用频率最少的键。