# 0. 前言


互联网上有很多不错的关于介绍Leveldb原理以及实现的博客，给了我很多帮助，这里就不一一列举了。 我想用另外一种风格或者方式，描述
我对leveldb的理解。以文字的形式，记录内心所知所想，有助于帮助加深理解。 我愿意作这样尝试，以期望帮助更多人。

Leveldb 是Google公司开源的存储引擎。 该存储引擎是基于日志结构的合并树[LSM-Tree](www)实现的, 优化了随机写，对于随机读并未作明显的优化。随着SSD
盘的广泛应用，硬件的随机读性能也有大幅提升。那么，SSD盘加上良好写性能的leveldb是一个不错的选择。

Leveldb基于LSM-Tree实现。在定义了日志格式，磁盘文件格式的基础上，实现了内存数据写入磁盘的Dump机制，去除冗余数据的Compaction机制，
内存数据重建与恢复功能，以及读写功能。图-1展示了leveldb的一个概貌。

![leveldb abstraction](http://i.imgur.com/1sd1LDP.png)

本文将按图-1圆圈的分布来组织章节结构：
* 第一章 LSM-Tree以及Leveldb的简介；
* 第二章 描述leveldb的日志格式，磁盘文件格式，内存数据格式，Bloom Filter 以及 Iterator实现；
* 第三章 分析leveldb的MemTalbe 写入磁盘的过程，Compaction实现，Version和Version的管理，以及Snapshot实现；
* 第四章 描述leveldb暴露给用户的主要接口；
* 第五章 详细描述bulkwrite的实现；
* 第六章 结束。
