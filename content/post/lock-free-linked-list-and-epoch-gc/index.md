---
title: "代码杂谈：无锁编程（一）"
description: 
date: 2021-11-28T20:44:45+08:00
slug: lock-free-linked-list-and-epoch-gc
image: img/card.png
draft: true
categories: ["Lock-Free Programming"]
tags: ["lock free linked list", "epoch gc" ]
---

## 缘起

俗话说的好，作为一个在数据库团队做 Kubernetes 开发的同学，不能做到编写 B 树至少要做到心里有 B 数。因此最近闲暇时间一直在研究可持久化的 B 树怎么写，接着就了解到了 BwTree（微软 14 年提出的无锁 B 树），然后开始思考其中的 mapping table 怎么实现，然后开始看 FasterKV（微软 18 年提出的可持久化无锁 KV），然后就彻底跑偏了...

本文主要记录我拿 Rust 折腾几个无锁数据结构的经验和教训，这样以后不记得了还能翻一翻，并告诉自己：

> 没事别瞎折腾什么劳什子无锁编程！

## 数据结构

通常，无锁数据结构的实现都要求有一些特殊的硬件支持，例如支持原子 compare-and-swap (CAS) 操作的处理器，或是[事务型内存（transactional memory）](https://en.wikipedia.org/wiki/Transactional_memory)。目前，主流的处理器都已经支持了 CAS 操作，因此大部分无锁数据结构是基于 CAS 操作设计的。通常来说，CAS 操作只能作用于 32/64 位整数，正好放下一个指针，因此各种无锁结构都围绕着指针的原子操作而设计。本文也将主要介绍一种无锁链表在 Rust 中的实现，链表算法主要基于 Zhang et al. 在 2013 年提出的一种无锁无序链表 [1]。

### 链表

首先介绍一下上述提到的无锁链表，它其实是一个结构为单向链表的无序集合，包含以下 3 种操作：

+ `insert(k)`，插入一个元素 `k` 并返回是否成功；
+ `remove(k)`，删除一个元素 `k` 并返回是否成功；
+ `contains(k)`，判断一个元素 `k` 是否在集合（链表）中；

链表的节点和整体结构如下图所示：

![Lock Free Linked List](img/linked-set-structure-and-operations.svg)

其中每个节点除了包含本身的元素以外，还有两个原子变量：

+ state，代表了当前节点的状态，论文中状态的定义有 4 种：
    + DAT，表示可见的元素（节点）
    + INS，表示插入中的元素
    + REM，表示删除中的元素
    + INV，表示无效的元素（节点）
+ next，存储了下一个节点的指针

和通常的单向链表一样，链表有一个 `head` 指针指向链表头，然后通过节点上 `next` 中存储的指针串联起来。当需要插入一个新节点的时候，使用 CAS 原子操作将 `head` 修改为新节点的地址，如图所示。当然 CAS 操作是可能会失败的，此时我们只需要重新进行图中的步骤。Rust 代码大致如下：

```rust
fn enlist(&self, node_ptr: SharedNodePtr<E>, g: &Guard) {
    debug_assert!(!node_ptr.is_null());
    let node = unsafe { node_ptr.deref() };
    let mut head_ptr = self.head.load(Ordering::Acquire, g);
    loop {
        // Set node.next to untagged(head).
        node.next.store(head_ptr.with_tag(0), Ordering::Release);   
        match self.head.compare_exchange(
            // Keep head's tag.
            head_ptr, node_ptr.with_tag(head_ptr.tag()),                      
            Ordering::AcqRel, Ordering::Acquire, g
        ) {
            Ok(_) => return,
            Err(err) => head_ptr = err.current,
        }
    }
}
```

在 3 种操作中，`contains` 是一个只读操作，因此实现它只需要按照链表无脑向下寻找就可，直到找到第一个可见的元素。在不考虑 GC 的情况下，只读操作永远都能正常运行。另外两个操作是比较类似的：它们都在链表头插入了一个包含了对应元素的节点，然后从该节点开始往后探查是否操作成功。算法流程大致为：

+ `insert` 插入状态为 `INS` 的节点，并向后找到第一个元素相同的节点
    + 节点状态为 `INS` 或者 `DAT` 则表明链表中已经存在该元素了，表明插入失败；
    + 节点状态为 `REM` 则表明链表中该元素已经被删除了，表明插入成功；
    + 节点状态为 `INV` 的忽略；
    + 未找到，则插入成功；
+ `remove` 插入状态为 `REM` 的节点，并向后找到第一个元素相同的节点
    + 节点状态为 `REM` 则表明链表中该元素已经被删除了，表明删除失败；
    + 节点状态为 `INS` 则表明有另一个线程正在插入该元素，尝试使用 CAS 标记为 `REM`；
        + 成功，表明删除成功；
        + 失败，则重新读取节点状态重试；
    + 节点状态为 `INV` 的忽略；
    + 节点状态为 `DAT` 则尝试用 CAS 标记为 `REM`，CAS 的成功/失败代表删除的成功/失败；

在插入和删除的向后查找过程中，如果遇到 `INV` 的无效节点，则会尝试使用 CAS 删除该节点，从而达到缩短链表的目的。

```rust
CAS(prev.next, cur_ptr, cur.next)
```


  

### 哈希表

## 垃圾回收（GC)

### Epoch Based


## 代码



## 参考文献

[1] Zhang, Kunlong, et al. "Practical non-blocking unordered lists." International Symposium on Distributed Computing. Springer, Berlin, Heidelberg, 2013.

[2] Harris, Timothy L. "A pragmatic implementation of non-blocking linked-lists." International Symposium on Distributed Computing. Springer, Berlin, Heidelberg, 2001.

[3] Chandramouli, Badrish, et al. "Faster: A concurrent key-value store with in-place updates." Proceedings of the 2018 International Conference on Management of Data. 2018.

[4] <https://github.com/crossbeam-rs/crossbeam>
