---
title: "2017 Alibaba Middleware 24h Final (Just for Fun 😀)"
date: 2017-07-26T16:59:26+08:00
tags: ["java", "distributed system", "parallel computing"]
categories: ["Database", "Contest"]
slug: awrace-topkn
---

今年阿里中间件比赛的时候不巧博主心情不好，外加要准备期末考试，并没有参加，非常遗憾。不过好在好基友 [Eric Fu](https://ericfu.me) 参加并获得了冠军！今年的主题是分布式数据库，如果想了解详情及复赛的解题思路请读者前往 Eric 的博客。

博主没有参加甚是遗憾，外加看到题目手痒难耐，遂问基友讨了最后的24h极客赛来玩一玩。

## 24h TOPKN

题目是分布式数据库上的分页排序，对应的SQL执行为 order by id limit k，n；主要的技术挑战为"分布式"的策略，赛题中使用多个文件模拟多个数据分片。

简称 top(k, n)。

<!--more-->
 
> 给定一批数据，求解按顺序从小到大，顺序排名从第k下标序号之后连续的n个数据 (类似于数据库的order by asc + limit k,n语义)

> top(k,3)代表获取排名序号为k+1,k+2,k+3的内容,例:top(10,3)代表序号为11、12、13的内容,top(0,3)代表序号为1、2、3的内容
需要考虑k值几种情况，k值比较小或者特别大的情况，比如k=1,000,000,000
对应k,n取值范围： 0 <= k < 2^63 和 0 < n < 100
数据全部是文本和数字结合的数据，排序的时候按照整个字符串的长度来排序。例如一个有序的排序如下(相同长度情况下ascii码小的排在前面)：
例： a ab abc def abc123

> 例如给定的k,n为(2,2)，则上面的数据需要的答案为: abc def

原题目在 [https://code.aliyun.com/wanshao/topkn_final](https://code.aliyun.com/wanshao/topkn_final)。

要求在24小时内完成，然而 ...

Codes 托管在 github 上 [https://github.com/arkbriar/topKN](https://github.com/arkbriar/topKN)，这个版本从基友的版本轻度修改而来 ...

## 解题过程


**主要条件** | | **主要限制**
:---- | :-:|:------
1. 3台机器: 2台worker，1台master <br /> 2. 输出在master上 | |1. Timeout: 5min <br /> 2. JVM heap size: 3G <br/> 3. 不允许使用堆外内存(FileChannel)


<br />
题目中一共有大约160000000条数据(字符串)，并分布在两台机器上，要进行 top(k, n)，首先 brute force 一定是不行的。

如果要对160000000条数据全排序JVM，第一内存太小，第二时间不够。对 10000000 条数据进行排序大约耗时为 1min13s，参考自 [https://codereview.stackexchange.com/questions/67387/quicksort-implementation-seems-too-slow-with-large-data](https://codereview.stackexchange.com/questions/67387/quicksort-implementation-seems-too-slow-with-large-data)；如果要进行外排序，磁盘读写导致耗时更长；以及还有最后要输出到master上，还有网络开销。

稍微了解一些数据库的同学都知道，数据库里有个叫索引(Index)的东西，是用来加速查询的。那我们首先来看一下 MySQL 的索引。

### MySQL 的索引

索引对查询的速度有至关重要的影响，理解索引也是进行数据库性能调优的起点。假设数据库中一个表有100000条记录，DBMS的页面大小为4K，并存储100条记录。如果没有索引，查询将对整个表进行扫描，最坏的情况下，如果所有数据页都不在内存，需要读取10000个页面，如果这10000个页面在磁盘上随机分布，需要进行10000次I/O，假设磁盘每次I/O时间为10ms，则总共需要100s(实际远不止)。但是如果建立 B+ 树索引，则只需要进行 $log_{100}1000000 = 3$ 次页面读取，最坏情况下耗时30ms，这就是索引带来的效果。

MySQL 有两种索引的类型：B+树和Hash索引。

参考自 [http://www.cnblogs.com/hustcat/archive/2009/10/28/1591648.html](http://www.cnblogs.com/hustcat/archive/2009/10/28/1591648.html)

### B/B+ 树

我们都知道RB-Tree和AVL-Tree是都是自平衡的二叉查找树，在这样的树上进行插入，删除和查询操作最坏情况都能在 $O(log_{2}n)$ 的时间内完成。

B树是1972年由 Rudolf Bayer 和 Edward M.McCreight 提出的，它是一种泛化的自平衡树，每个节点可以有多于两个的子节点。与自平衡二叉查找树不同的是，B树为系统大块数据的读写操作做了优化。B树减少定位记录时所经历的中间过程，从而加快存取速度。所以B树常被应用于数据库和文件系统的实现。

假设B树每个节点下有b个子节点(b个分块)，那么查找可以在约为logbN的磁盘读取开销下完成。

关于 B/B+ 树的其他特征在维基和其他资料上有详细讲解，在此不再赘述。

由于时间限定的缘故，实现 B/B+ 树的索引显得不太现实，所以我们将考虑一种退化的方式 —— 桶排序。

参考自 

1. [https://zh.wikipedia.org/wiki/B%E6%A0%91](https://zh.wikipedia.org/wiki/B%E6%A0%91)
2. [https://zh.wikipedia.org/wiki/B%2B%E6%A0%91](https://zh.wikipedia.org/wiki/B%2B%E6%A0%91)


### 桶排序

先来看一下数据的分布情况：

> 每一行字符串的长度为 [1,128] （数字在Long值范围内均匀分布）

显然以字符串长来标识桶是可行的，但是我们有160000000条字符串，128个桶每个桶还是有1250000条，这样的桶粒度还是太粗。

由于我们手里没有其他分布情况，以及字符串大小在同长度下是字典序，所以以字符串长度+前2-3个字符作为桶的标识就可以了。

在本题中，由于只进行5轮query，并且写开销非常大，故仅对桶内字符串的数目进行统计。

### 基本思路

1. Worker 分别扫描并建立所有桶的计数
2. Master 汇总所有桶，并建立全局桶
3. Master 通过二分查找，找到(k, n)所在的几个桶
4. Master 向 Worker 请求获得这些桶内的所有字符串
5. Master 对这些字符串进行排序，并输出最终结果

## 实现过程

本题实现的最大难点在于 I/O 和 GC 调优，还有充分调动 CPU。

1. GC 调优我们通过使用固定的 buffer 来进行操作，基本避免出现 GC。
2. 通过多线程处理来保证 CPU 调动。
3. 网络开销在上述算法下基本忽略了

然而，剩下的 I/O 调优让人非常头大，并伴随着可能存在的同步问题 (读和处理)

我们来看一下硬件条件：

1. CPU: 24 core, 2.2GHz
2. 内存: 3G heap memory
3. 数据磁盘：内存文件系统
4. 临时、结果磁盘：SSD

### Top 1 的最终结果

从基友那得知，top 1 的实现5轮总耗时(JVM启动、暂停，以及计算)共11s多，用来推算自己离最优还有多远。

(我实在没想通第一名怎么跑到11s的，大概是我内存文件系统开的不对？)

### 不稳定的I/O实现

每个文件单线程读，固定4线程处理，通过预先分配的 buffer 避免出现 GC。

封装文件操作并行，5个文件并行总共占用 25 core。

实现下来由于读和操作无法有效平衡，很难做到读完就操作，大量时间花费在等待读线程新产出一个buffer或是处理线程处理完一个buffer。

在清理系统cache的情况下，大约8s处理完所有字符串，不清理大约为4s。

这种实现无法调优，等待的时间完全看 OS 心情。

### 稳定的I/O实现

其实之前提到过，单线程读并不是一个很好的思路，而且上述的处理架构存在着不稳定的等待问题。

那么如何使得在保持多线程读的同时能够多线程处理，并且不浪费时间在等待上呢？

答案是**每一个线程在读完一份数据后，直接进行处理，然后读取下一份**。

但是如何安排每一个线程的那份数据，保证线程之间不会有重复读取，也不会有漏读的数据块？

这里采用 Eric 在复赛里的实现: FileSegment, 通过将一个文件分割为一个一个 Segment，让每一个线程自己请求下一个 Segment，并通过 RandomAccessFile 进行读取。(由于不许用 FileChannel)

```java
public class FileSegment {
    private File file;

    private long offset;

    private long size;
    ...
}
```

```java
public abstract class BufferedFileSegmentReadProcessor implements Runnable {
    private final int bufferMargin = 1024;
    private FileSegmentLoader fileSegmentLoader;
    private int bufferSize;
    private byte[] readBuffer;

    // Here bufferSize must be greater than segments' size got from fileSegmentLoader, 
    // otherwise segment will not be fully read/processed!
    public BufferedFileSegmentReadProcessor(FileSegmentLoader fileSegmentLoader, int bufferSize) {
        this.bufferSize = bufferSize;
        this.fileSegmentLoader = fileSegmentLoader;
    }

    private int readSegment(FileSegment segment, byte[] readBuffer) throws IOException {
        int limit = 0;
        try (RandomAccessFile randomAccessFile = new RandomAccessFile(segment.getFile(), "r")) {
            randomAccessFile.seek(segment.getOffset());
            limit = randomAccessFile.read(readBuffer, 0, readBuffer.length);
        }
        return limit;
    }

    protected abstract void processSegment(FileSegment segment, byte[] readBuffer, int limit);

    @Override
    public void run() {
        readBuffer = new byte[bufferSize + bufferMargin];
        try {
            FileSegment segment;
            while ((segment = fileSegmentLoader.nextFileSegment()) != null) {
                int limit = readSegment(segment, readBuffer);
                processSegment(segment, readBuffer, limit);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

这样每一个线程都在读和处理之间循环，先完成的线程会请求下一个 Segment，保证以最短的时间完成调用所有 Core 进行一遍数据处理。

这个方法保证能够十分稳定的进行读取和处理，并且不需要等待 Reader 和 Processor 通信，在我的实验环境中基本存着cache 3-4s能扫描一遍，清理cache也在 7-8s。

其实到这里本题已经差不多了，我记得 Eric 在这种处理下比赛的最终结果是 14s 多。

### Top 1 的优化

1. 在建立桶的时候，同时将每个文件上第i条字符串对应的offset、桶序号存入两个数组，并将桶信息以及这两个数组一起存储到本地，这样构成了一个可以快速查询桶内所有字符串的索引
2. 这些信息总计大约 1G 左右，通过建立这样的索引，每次仅讲索引信息读入，并进行有限次的读取，就可以获取到所有需要的字符串

对比之下，之前的方式是从内存文件系统读取5G的文件，现在是从SSD读取1G左右文件，只要并发读速度差距不在5倍以上，这样的优化是能节省不少时间！

## 测试

仅测试了建 Index

### 清理 cache

```bash
$ sysctl -w vm.drop_caches=3
vm.drop_caches = 3
$ time java ${JAVA_OPTS} -cp /exp/topkn/topkn.jar com.alibaba.middleware.topkn.TopknWorker localhost 5527 .
[2017-07-27 14:24:27.512] INFO com.alibaba.middleware.topkn.TopknWorker Connecting to master at localhost:5527
[2017-07-27 14:24:27.519] INFO com.alibaba.middleware.topkn.TopknWorker Building index ...
0.673: [GC (Allocation Failure) 0.673: [ParNew: 809135K->99738K(943744K), 0.1759305 secs] 809135K->542134K(3040896K), 0.1761763 secs] [Times: user=3.15 sys=0.69, real=0.17 secs]
[2017-07-27 14:24:32.330] INFO com.alibaba.middleware.topkn.TopknWorker Index built.
Heap
 par new generation   total 943744K, used 903716K [0x0000000700000000, 0x0000000740000000, 0x0000000740000000)
  eden space 838912K,  95% used [0x0000000700000000, 0x00000007311225e0, 0x0000000733340000)
  from space 104832K,  95% used [0x00000007399a0000, 0x000000073fb06b38, 0x0000000740000000)
  to   space 104832K,   0% used [0x0000000733340000, 0x0000000733340000, 0x00000007399a0000)
 concurrent mark-sweep generation total 2097152K, used 442395K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 3980K, capacity 4638K, committed 4864K, reserved 1056768K
  class space    used 434K, capacity 462K, committed 512K, reserved 1048576K

real    0m5.238s
user    1m11.980s
sys     0m26.324s
```

### 不清理 cache

```bash
$ time java ${JAVA_OPTS} -cp /exp/topkn/topkn.jar com.alibaba.middleware.topkn.TopknWorker localhost 5527 .
[2017-07-27 14:25:39.986] INFO com.alibaba.middleware.topkn.TopknWorker Connecting to master at localhost:5527
[2017-07-27 14:25:39.992] INFO com.alibaba.middleware.topkn.TopknWorker Building index ...
0.590: [GC (Allocation Failure) 0.590: [ParNew: 838912K->99751K(943744K), 0.1851592 secs] 838912K->591302K(3040896K), 0.1855308 secs] [Times: user=3.03 sys=0.88, real=0.19 secs]
[2017-07-27 14:25:44.449] INFO com.alibaba.middleware.topkn.TopknWorker Index built.
Heap
 par new generation   total 943744K, used 863664K [0x0000000700000000, 0x0000000740000000, 0x0000000740000000)
  eden space 838912K,  91% used [0x0000000700000000, 0x000000072ea02318, 0x0000000733340000)
  from space 104832K,  95% used [0x00000007399a0000, 0x000000073fb09e00, 0x0000000740000000)
  to   space 104832K,   0% used [0x0000000733340000, 0x0000000733340000, 0x00000007399a0000)
 concurrent mark-sweep generation total 2097152K, used 491550K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 3980K, capacity 4638K, committed 4864K, reserved 1056768K
  class space    used 434K, capacity 462K, committed 512K, reserved 1048576K

real    0m4.754s
user    0m57.144s
sys     0m30.568s
```

**UPDATE**

在我的8核小物理机上，跑一边内存文件系统里的数据不要1s... 看来是机器的问题，云服务器还是不给力啊 🤣

## 总结

今年的中间件比赛是在很有意思，这个题目对参赛选手有比较高的要求，解题中涉及到多线程并发读写、分布式排序、索引、Socket编程、JVM 调优等多个方面，我这种上来直接挑的果断挂了两个版本的代码，东看西看左问右问终于把最终形式给弄清楚了。

说是我做的不如说是我在学如何做，在此过程中发现了很多有意义的技术点，正准备接下来继续学习。


