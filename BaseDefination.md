# 2. 基本数据结构的定义

本章依次介绍Leveldb中的日志文件格式定义，磁盘文件（SST）格式定义，以及其他的一些基础数据结构的定义。

## 2.1 日志文件格式

在Leveldb中，日志被顺序写入文件中。当需要回复数据的时候，扫描日志文件，读取每一条日志记录，恢复内存中的数据。在doc/log_format.txt文件中，有详细描述日志文件以及其格式定义。

日志文件由一系列32K大小的块（Block)构成，每个Block中依次存放日志条目（Record)，每一个Record包含7字节的头（Header)其余部分为日志内容。 若一个Block的剩余部分字节数小于7字节（Header的大小），则该剩余部分填充0处理。图2-1展示日志文件的格式，Block的布局，以及Record的布局。

![Log Format](http://i.imgur.com/xYio5k9.png)

图2-1 日志文件布局图

如图2-1所示，文件由Block构成，有的Block会结尾小于7字节的部分会填充0（图中绿色区域）。当然，有的Block在插入一条记录后刚好满了，那就不需要填0处理了。 如果这个Block刚好有7个字节，那就写入一个Record(显然只能写入Header)。

Block是由一系列Record构成。每一条Record由Header和Content部分组成，Header包含校验码（4字节），内容长度（2字节），类型信息（1字节），Content是用户写入的数据。 因为一个Block为32K，所以内容长度用2字节可以表示。若用户的数据超过32字节，或者一个Block剩下的空间无法容纳用户数据怎么办？这个时候超出一个Block的数据会被分成多个连续的Record记录在日志文件里。第一个Record的type 表示为FIRST，最后一个LAST, 中间的Record的type 值为MIDDLE。

这种日志文件格式的优点：
* 易于异常处理；出现异常情况是，忽略掉坏损的Block，处理下一个Block即可。
* 容易正确切分文件内容，按Block边界切分，比较适合MAPREDUCE处理；
* 无需额外的meta数据维护较大的日志内容；
