# 存储引擎系列



由于工作原因，我接触到[leveldb](https://code.google.com/p/leveldb/)，并花了一些时间试图了解其原理以及实现。随着工作的进展，原
生的leveldb已经无法满足我们的需求了。我们没有别的选择，需要对leveldb作二次开发。于是，又花了些时间，对其代码作了一次详细的梳
理，以加深对其理解。

或许工作过于繁杂，或许记忆力在悄然退步，对于一些曾很熟悉的细节，一段时间后总是有生疏感。 俗话说，好记性不让烂笔头。

随着对leveldb的了解，我欣喜地发现，Facebook对其作了很显著的改进。这进一步激起了我要搞清楚改进版的详细信息。 在搞清楚后，以文字方式展现出来是一项有意义，有挑战的事情。

...

有太多原因促使我坚持写下去。

## Leveldb篇

Contents:
* [0. 前言](https://github.com/fengmao/notes/blob/master/xproject/Introduction.md)
* 1. LSM-Tree模型(目前还没想好从那个角度展开)
* [2. 基本数据结构](https://github.com/fengmao/notes/blob/master/xproject/BaseDefination.md)
  * [2.1 日志文件格式](https://github.com/fengmao/notes/blob/master/xproject/BaseDefination.md#21-%E6%97%A5%E5%BF%97%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F)
  * [2.2 磁盘文件(SST)布局](https://github.com/fengmao/notes/blob/master/xproject/BaseDefination.md#22-%E7%A3%81%E7%9B%98%E6%96%87%E4%BB%B6sst%E5%B8%83%E5%B1%80)
  * [2.3 构建磁盘文件(SST)](https://github.com/fengmao/notes/blob/master/xproject/BaseDefination.md#23-%E7%A3%81%E7%9B%98%E6%96%87%E4%BB%B6sst%E6%9E%84%E5%BB%BA)
  * [2.4 读取磁盘文件(SST)](https://github.com/fengmao/notes/blob/master/xproject/BaseDefination.md#24-%E8%AF%BB%E5%8F%96sst%E6%96%87%E4%BB%B6)
* [3. Compaction机制](https://github.com/fengmao/notes/blob/master/xproject/Compaction.md)
  * [3.1 文件组织结构](https://github.com/fengmao/notes/blob/master/xproject/Compaction.md#31-%E6%96%87%E4%BB%B6%E7%BB%84%E7%BB%87%E7%BB%93%E6%9E%84)
  * [3.2 Snapshot实现](https://github.com/fengmao/notes/blob/master/xproject/Compaction.md#32-snapshot%E5%AE%9E%E7%8E%B0)
  * [3.3 Compaction](https://github.com/fengmao/notes/blob/master/xproject/Compaction.md#33-compaction)
  * [3.4 Compaction触发的条件](https://github.com/fengmao/notes/blob/master/xproject/Compaction.md#34-compaction%E8%A7%A6%E5%8F%91%E7%9A%84%E6%9D%A1%E4%BB%B6)
  * [3.5 Compaction流程](https://github.com/fengmao/notes/blob/master/xproject/Compaction.md#35-compaction%E6%93%8D%E4%BD%9C%E6%B5%81%E7%A8%8B)
  * [3.6 Compaction选取文件](https://github.com/fengmao/notes/blob/master/xproject/Compaction.md#36-%E9%80%89%E5%8F%96%E5%8F%82%E4%B8%8Ecompaction%E6%93%8D%E4%BD%9C%E7%9A%84%E6%96%87%E4%BB%B6)
  * [3.7 Compaction过程详解](https://github.com/fengmao/notes/blob/master/xproject/Compaction.md#37-compaction%E6%93%8D%E4%BD%9C%E7%9A%84%E8%AF%A6%E8%A7%A3)
  * [3.8 Compaction操作后续处理](https://github.com/fengmao/notes/blob/master/xproject/Compaction.md#38-compaction%E6%93%8D%E4%BD%9C%E8%BE%93%E5%87%BA%E6%96%87%E4%BB%B6)
  * [3.9 小结](https://github.com/fengmao/notes/blob/master/xproject/Compaction.md#39-%E5%B0%8F%E7%BB%93)

* [4. 读数据操作分析](www.undefined.com)
* [5. 写数据操作分析](www.undefined.com)
* [6. Bulk Write操作分析](www.undefined.com)
* [7. 结束](www.undefined.com)
* [附录](www.undefined.com)
2014-05-07 于浙江图书馆 丰茂

