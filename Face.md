# 存储引擎系列



由于工作原因，我接触到[leveldb](https://code.google.com/p/leveldb/)，并花了一些时间试图了解其原理以及实现。随着工作的进展，原
生的leveldb已经无法满足我们的需求了。我们没有别的选择，需要对leveldb作二次开发。于是，又花了些时间，对其代码作了一次详细的梳
理，以加深对其理解。

或许工作过于繁杂，或许记忆力在悄然退步，对于一些曾很熟悉的细节，一段时间后总是有生疏感。

随着对leveldb的了解，我欣喜地发现，Facebook对其作了很显著的改进。这进一步激起了我要搞清楚改进版的详细信息。

以上是为序。

## Leveldb篇

Contents:
* [0. 前言](https://github.com/fengmao/notes/blob/master/Introduction.md)
* [1. LSM-Tree以及Leveldb简介](https://github.com/fengmao/notes/blob/master/Introduction.md)


2014-05-07 于浙江图书馆 丰茂
