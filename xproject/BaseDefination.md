# 2. 基本数据结构的定义

本章依次介绍Leveldb中的日志文件格式定义，磁盘文件（SST）格式定义，以及其他的一些基础数据结构的定义。

## 2.1 日志文件格式

在Leveldb中，日志被顺序写入文件中。当需要回复数据的时候，扫描日志文件，读取每一条日志记录，恢复内存中的数据。在doc/log_format.txt文件中，有详细描述日志文件以及其格式定义。

**日志文件布局**

日志文件由一系列32K大小的块（Block)构成，每个Block中依次存放日志条目（Record)，每一个Record包含7字节的头（Header)其余部分为日志内容。 若一个Block的剩余部分字节数小于7字节（Header的大小），则该剩余部分填充0处理。图2-1展示日志文件的格式，Block的布局，以及Record的布局。

![Log Format](http://i.imgur.com/xYio5k9.png)

图2-1 日志文件布局图

如图2-1所示，文件由Block构成，有的Block会结尾小于7字节的部分会填充0（图中绿色区域）。当然，有的Block在插入一条记录后刚好满了，那就不需要填0处理了。 如果这个Block刚好有7个字节，那就写入一个Record(显然只能写入Header)。

Block是由一系列Record构成。每一条Record由Header和Content部分组成，Header包含校验码（4字节），内容长度（2字节），类型信息（1字节），Content是用户写入的数据。 因为一个Block为32K，所以内容长度用2字节可以表示。若用户的数据超过32字节，或者一个Block剩下的空间无法容纳用户数据怎么办？这个时候超出一个Block的数据会被分成多个连续的Record记录在日志文件里。第一个Record的type 表示为FIRST，最后一个LAST, 中间的Record的type 值为MIDDLE。

这种日志文件格式的优点：
* 易于异常处理；出现异常情况是，忽略掉坏损的Block，处理下一个Block即可。
* 容易正确切分文件内容，按Block边界切分，比较适合MAPREDUCE处理；
* 无需额外的meta数据维护较大的日志内容；

这种实现也有不足之处：
* 对Record没有Pack机制；
* 对Block没有作数据压缩；

## 2.2 磁盘文件（SST）的格式

### 2.2.1 磁盘文件布局

在Leveldb中，数据最终以SST文件形式存储在磁盘中的。在doc/table_format.txt文件中有记录该文件的一些信息。SST文件以4K大小的块(Block)构成。这些Block按其存储的数据可以分为 数据块(Data Block), 元数据(Meta Data Block), 索引数据块(Index Block)。在文件的结尾存成一个固定长度的Footer, 包含元数据索引句柄，数据索引句柄，或许有填充数据，Magic Code。图2-2展示了SST文件的整体布局。

![](http://i.imgur.com/UVLVK2V.png)

图2-2 SST Layout

**Data Block 构建**

在SST文件中，目前有2中类型的Block, 一种是数据Block， 存储用户数据，如图2-2中的Data Block Layout所示；另外一种是Meta Data Block。两种Block的区别主要在于索引的不同。对于Data Block而言，并不是每一个Data Entry都会创建索引，而是隔一定数量的Data Entry，才对这个Entry创建索引。索引是一个Map, 其Key就是被索引Data Entry的key, 其Value是这个Data Entry在文件中的Offset. 这个Map就以数组的形式存储在Block结尾处。Map中的每一项内容称之为RestartPoint。 

在leveldb中，Data Block主要是存储用户数据，和存储Block的索引数据。Block中的数据是按照Key严格有序的。那Data Block是如何构造出来的呢？ 只有在内存table dump到磁盘，或者在compaction的过程中，会创建SST文件，只有在构造SST 文件的过程中才会构造Data Block。 上述2个场景的数据都是有序的。

由于为了节约存储空间，Data Block中的Data Entry采用前缀压缩的方式存储。具体而言，若第i+1个DataEntry 与 第i个Data Entry的Key相似部分的话，第i + 1个Data Entry不需要存储相同的内容，只存储不同部分的内容。但是，被索引的Data Entry需要存储完整的Key。考虑到Key是有序的，相邻Key拥有相同内容是比较常见的。这种组织方式能够节约存储空间。图2-3展示了一个普通的Data Block的构建过程：

* 输入一个Key-Value集合，按Key有序；
* 构建Data Entry(将key前缀压缩，和value 拼在一起，并在最前面拼接key, value的长度信息）；
* 若需要创建Restart Point，将当前Key的在block中的offset存放到restart point 数组中；
* 将构建好的Data Entry依次写入文件；
* 将Restart Point数组中的数字依次写入文件（每个元素4字节，不压缩）；
* 将Restart Point数组的长度写入文件；
* 填充Trailer数据写入文件（Trailer数据长度固定）。


![Block Construct](http://i.imgur.com/jgF8KdB.png)


**Data Block读取** 

在leveldb中，读取操作都是通过Iterator来进行的。使用Iterator的好处是，可以提供统一的接口，保持代码风格一致；还可以同时拥有批量和随机读取的功能。可谓一举多得。

其实，明白Data Block的布局后，很容易想到如何实现随机和批量读取Data Block的接口了。读取Block之前，我们需要获取Block的句柄，也就是预读取的Block在文件中的Offset 以及 Size。在构建Data Block时候，在Block的末尾添加了固定字节数（5字节）的Trailer数据。通过如下步骤，可以把Block的Data Entry读入内存，并读出Restart Point数组：

* 从文件中读取Size + 5 bytes内容到内存；
* 解析上述内容的最后5字节，获取CRC以及压缩类型，判断数据是否损坏，或者需要解压缩；
* 获得Block的内容后，将最后4字节解析为Restart Point Count, 获得RestartPoint 数组的长度；
* 根据Restart Point 数组的长度，计算Restart Point数组的起始地址（数组元素是4字节）；
* 根据上面步骤，可以确定Data Entry的区域，以及Restart Point数组区域；

接下来我们看如何随机读取一个Key。 假设我们预读取的key 为target. 那么我们可以利用RestartPoint数组进行二分查找，定位到目标Key的前一个RestartPointKey，然后通过线性查找，定位到目标Key；具体过程可以这样描述：

* 首先将Restart Point数组N/2 - 1处元素对应的Key 与目标key进行比较；
* 若比N/2 - 1处的Key大于目标Key，则在0 ~ N/2 - 1之间重复上述查找；
* 否则在N/2 - 1 ~ N 之间重复上述查找；

上述步骤的结果是，找到离目标Key最近的一个Restart Point Key. 那么接下来，从这个Restart Point key开始，挨个比较，直到目标Key存在，或者目标Key不存在。

**Meta Data Block**

对于元数据块（Meta Data Block）而言，建立索引的方式有明显不不同，是对Block中的每一个Data Entry都建立索引。这种Block我们可以理解为，采用自然数0,1,2 .. 作为Key，并不存储在Block中（逻辑上的Key), 每个Entry的均建有索引，存储在Block的结尾处。索引数据是一个数组，数组元素是对应的Data Entry在文件中的Offset. 若要查找第i个Data Entry， 通过索引（数组下标）中的数据，得到具体的Data Entry。相对Data Block而言，这种索引查找效率高，占用更多的存储空间，适合存储元数据。

**Meta Index Block**

在SST文件中，接近这数据Block存放的是Meta Data Block, 而文件可能存在多个Meta Data Block, 为了能够快速检索出具体的Meta Data Block, SST文件对Meta Data Block作了索引。索引数据存储在一个Data Block中，其Key是Meta Data Block的名称，其Value是一个Block句柄，该句柄记录被索引的Block在文件中的Offset 和 Size。根据这个句柄，能够读出具体Block的数据。

在目前的实现中，整个文件只有一个Meta Data Block, 所以Meta Index Block中只有一条记录。

**Index Block**

SST文件为每一个Block创建了索引。这些索引数据存储为一个Data Block, 位于Meta Index Block 结尾处。索引数据组织成一个Key-Value 表，其Key是一个不小于Data Block中的最大Key(显然是Data Block的最后一个Key)，并且小于相邻的下一个Block的最小Key 。为了节省空间，在创建SST文件的时候，会在前一个文件的最大Key 与 相邻文件的最小Key之间选择一个“最短”的Key, 这样能够节省存储空间。索引表的Value是一个Block句柄。

通常，将SST的Index Block 缓存在Cache中，这样能够加快Search Key的过程。

**Footer**

在SST文件的结尾，存储了一段特殊的，固定长度的数据，称之为Footer。这段固定长度的数据中，依次记录了Meta Index Block的句柄，Index Block的句柄，填充数据（如果需要的话），Magic Code。其布局见图2-2 Footer Layout。


## 2.4 磁盘文件（SST）构建

在分析了SST布局，以及Data Block的构建和读取后，接下来就比较容易说明如何构建SST文件了。 在leveldb中，需要构建SST文件的场景有：

* 内存中有序的Key-Value需要Dump 到level 0;
* 在Compaction的场景中，多个文件合并生成一个新的SST文件。


在SST文件的构建过程中，用户是通过接口将有序的Key-Value依次加入到文件中，到文件尺寸或者Key-Value处理完时，调用Finish表示构建文件完成了。实际上，具体的构建过程比较有趣：

* 1)  用户调用Add接口添加一个Key-Value;
* 2)  若当前的Data Block未满，则将1）中的Key-Value添加到当前Data Block中；
* 3)  若当前的Data Block已经满了，则：
      * a) 为根据当前Block的所有Key调用filter-policy创建一个filter,并把filter的内容加到metablock中；
      * b) 根据当前Block的最后一个key（称为A), 和当前输入的key(称为B），结合Comparator找到一个比A大，且比B小的X，
      * c) 将{X, 当前Block在文件中的offset 和 size} 作为Key-Value添加到Index Block中；
* 4)  重复上述步骤，直到所有key-value处理完毕，或者文件大小超过阈值才结束；
* 5)  为meta block创建索引数据，写入meta index block;
* 6)  将meta block 写入文件；
* 7)  将meta index block写入文件；
* 8)  将index block写入文件；
* 6)  构建Footer，填充meta 相关的值，并写入文件；


![SST Construction](http://i.imgur.com/x4rhV80.png)

图2-4 SST文件的构造示意图
 


 
