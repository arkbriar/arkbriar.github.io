---
title: "代码杂谈：无锁编程（一）"
description: 
date: 2021-11-28T20:44:45+08:00
slug: lock-free-linked-list
image: img/card.png
draft: true
categories: ["Lock-Free Programming"]
tags: ["lock free linked list", "epoch framework" ]
---

## 缘起

俗话说的好，作为一个在数据库团队做 Kubernetes 开发的同学，不能做到编写 B 树至少要做到心里有 B 数。因此最近闲暇时间一直在研究可持久化的 B 树怎么写，接着就了解到了 BwTree（微软 14 年提出的无锁 B 树），然后开始思考其中的 mapping table 怎么实现，然后开始看 FasterKV（微软 18 年提出的可持久化无锁 KV），然后就彻底跑偏了...

本文主要记录我拿 Rust 折腾几个无锁数据结构的经验和教训，这样以后不记得了还能翻一翻，并告诉自己：

> 没事别瞎折腾什么劳什子无锁编程！

## 数据结构

### 链表

### 哈希表

## 垃圾回收（GC)

### Epoch Based


## 代码



## 参考文献

[1] Zhang, Kunlong, et al. "Practical non-blocking unordered lists." International Symposium on Distributed Computing. Springer, Berlin, Heidelberg, 2013.

[2] Harris, Timothy L. "A pragmatic implementation of non-blocking linked-lists." International Symposium on Distributed Computing. Springer, Berlin, Heidelberg, 2001.

[3] Chandramouli, Badrish, et al. "Faster: A concurrent key-value store with in-place updates." Proceedings of the 2018 International Conference on Management of Data. 2018.

[4] <https://github.com/crossbeam-rs/crossbeam>
