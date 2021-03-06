---
layout: post
title: HDFS:Hadoop Distributed File System简介
description: HDFS是一个文件系统，为什么会存在文件系统？HDFS是分布式的文件系统，它又有什么特别的实现机制吗？
category: Hadoop
---

##背景

最近要做一件事：调研远端节点向HDFS上传/下载文件的方法，并确定一个高效率的方案，这个最终方案初步确定了WebHDFS REST API方式，不过一些HDFS内部的实现细节/原理，自己很想一窥究竟，OK，既然有这份心情，那就走起，趁热打铁、一鼓作气。

##文件系统

为了实现多个应用可以并发的向存储介质上写入信息，出现了文件（File）这一概念，即把信息以一种单元（File，文件）的形式存储在介质上。文件是一个命名的、存储在设备上的信息的线性字节流。对文件的管理，包括文件的逻辑组成结构、命名、操作（读、写、删除）、权限管理、具体实现（如何组织存储区域），称为`文件系统`（FileSystem，FS）。

总结一下，文件系统需要解决文件相关的几个问题：

1. 怎么找到文件？（文件的命名）
2. 可以对文件进行哪些操作？（读、写、删除、复制等）
3. 哪些人可以对这个文件进行操作？（权限管理）
4. 为保证上面的功能，并且，兼顾文件读写等操作的效率，逻辑上，一个文件可以划分为几个部分？（逻辑组成结构）
5. 说一千道一万，上面这些东西，如何在物理磁盘上实现？（具体实现）

无结构的文件 vs. 结构化的文件

无结构的文件：大部分文件系统，不关系文件里保存的数据，把文件内容作为无结构的字节序列保存。也有一些文件系统支持结构化的文件，以记录为单位组织信息，免去文件系统使用者将“原始的”字节流转换成记录流的麻烦。

一个文件，通常包含：文件名和文件数据，还包括，创建日期、文件长度、引用计数等额外信息，通常将这些额外项，称为文件属性，也称文件元数据。

###目录与目录树

如何找到一个文件呢？为了能够高效地找到文件，引入了“目录”这一概念。“目录”，又称“文件夹”，就是一个虚拟容器，其内部可能包含着一组文件和其他的目录（子目录）。目录和子目录，就形成了一个层次结构，称为目录树。

为什么要有目录？没有行不行？没有目录可以，但是文件命名就比较麻烦，因为没有目录，所有的文件都不能同名，哪怕是简单的 `index.html`、`default.css`等文件也都只能有一个，所以说，本质上，目录实现了文件命名空间的分割，不同命名空间内，可以有同名文件存在。

不同物理磁盘上，格式化得到的多个文件系统，可以相互挂载（嫁接），例如两个文件系统 FS1 和 FS2 ，将 FS2 挂载到 FS1 的 `/home` 目录下，则，可以通过 `/home/user1`
访问原来 FS2 上的 `/user1` 目录。有一点需要说明的是：原来 FS1 目录 `/home` 下的所有内容，都被隐藏而无法访问了，但是将 FS2 从 FS1 上卸载之后，仍能正常访问。

###文件系统的实现

文件，是逻辑上的概念，其实际上在物理磁盘上是如何存储的呢？


##常见问题

###小文件个数多

HDFS支持超大文件（几百MB、几百GB、甚至TB=1024GB、PB=1024TB的文件），是通过将数据分布在数据节点（DataNode），并且将文件的元数据（目录树结构、文件-数据块索引等信息）存放在名字节点（NameNode）上，来实现的。名字节点（NameNode）的内存大小，决定了HDFS可以保存的文件数量；虽然现在内存数量比较大，但大量的小文件仍然严重影响名字节点（NameNode）的性能。


##参考来源

* 《Hadoop技术内幕：深入解析Hadoop Common和HDFS架构的设计和实现原理》

[NingG]:    http://ningg.github.com  "NingG"
