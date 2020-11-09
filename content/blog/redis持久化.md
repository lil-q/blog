---
title: "Redis：持久化"
date: 2020-10-16T10:56:25+08:00
slug: ""
description: ""
description: "介绍Redis雪崩击穿穿透和两种持久化机制"
keywords: [Redis, 雪崩, 击穿, 穿透, 两种持久化机制]
tags: [Redis]
draft: true
math: false
toc: true
---

Redis 是内存数据库，内存中的数据一旦断电就会丢失，所以需要对 Redis 中的数据持久化。Redis 提供两种持久化方式：RDB 持久化和 AOF 持久化。

## 一、RDB 持久化

RDB 持久化既可以手动执行，也可以根据服务器配置选项定期执行，该功能可以将某个时间点上的数据库状态保存到一个 RDB 文件中。RDB 文件是一个经过压缩的二进制文件，通过该文件可以还原生成 RDB 文件时的数据库状态。

### 1.1 RDB 文件的创建和载入

Redis 中有两个命令可以生成 RDB 文件，*SAVE*