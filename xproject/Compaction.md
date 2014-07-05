
# 3. Compaction 机制

本章介绍Leveldb的Compaction机制。在此之前，需要介绍leveldb的文件组织结构，以及Snapshot的实现。明白这些后，更容易理解compaction中的一些细节问题。

## 3.1 文件组织结构

在leveldb中，任何一个时刻，文件被组织成为层次结构，有如下特点：
* 1) 每一个层次最多能够存储的数据量是有限制的, 更高层可以容纳更多数据量；
* 2) 除第0层外，其他每一层数据是有序的，文件与文件之间不存在Key范围交叠；
* 3) 系统维护当前文件层次结构的一个视图；

只有在Compaction或者，内存数据Dump到磁盘后，会改变文件的层次结构，系统创建一个该层次结构的一个全新的视图，替换现有视图。在leveldb中，这个文件层次结构的视图抽象为Version。 当有文件增加或减少的时候，会记录在VersionEdit中。通过当前Version 与 VersionEdit 中的信息，生成新的视图Version。系统会维护活跃的Version链表。当旧的Version上的所有读操作都结束了，那么该Version也就被释放了。

## 3.2 Snapshot实现

Snapshot是一种快照机制，使得读操作不受写操作的影响。在leveldb中，借助每一个Key的sequence完美的实现了该功能。

在leveldb中，每一个KV的KEY都有一个全局唯一的sequence。 当用户对某个时刻作了快照后，记录该时刻的最大的sequence。那么，后续读取这个snapshot的数据的时候，只要返回小于等于整个sequence 的KV即可。

事情还不是这么简单，因为leveldb有后台线程会适时作compaction操作，该操作会删除数据。显然，compaction操作不能删除对任何一个snapshot可见的KV。

显然，SNAPSHOT是有代价的。若系统长时间保留SNAPSHOT，会导致冗余数据不能尽快删除，读放大，某些KV多次compaction都无法删除，浪费磁盘IO以及CPU资源。 故在使用完SNAPSHOT后应该尽快释放。

## 3.3 Compaction

在leveldb中，所谓Compaction操作就是删除冗余数据的操作。Leveldb将对磁盘的随机写操作转化为一次写内存和一次顺序写磁盘操作。对某一个数Key-Value(下文简称KV)的更新，不是去定位到该KV，进行更新。而是，直接新写入新的KV。在读的时候，总是向用户返回最新写入的数据。那么，leveldb中就会存在一个KV的多个“版本”， 那么除了该KV的最进一次修改的版本外，其他的都是无效数据，可以删除了。这些无效数据会在特定的时候被系统删除掉。

Leveldb中有专门的后台线程执行Compaction操作。

## 3.4 Compaction触发的条件

在leveldb的Compaction实现中，有三种类型的Compation操作，分别是：Size Comacption, Seek Compaction 和  Manual Compaction。 其中，Manual Compaction操作以用户接口的形式提供，用户可随时调用，这里就不详细展开。

** Size Compaction **

Leveldb中的文件分层组织，每一次中允许存放的文件总的大小是有限的。所谓Size Compaction是指，当某一层文件总数据量超过阈值后，就会触发Compaction操作，来消除冗余数据，达到减少该层文件总数据量。当冗余数据减少了，就会有效减缓读放大程度。这种情况触发的Compaction称之为Size Compaction。

** Seek Compaction **

同样，由于Leveldb中的文件分层组织，读取某一个KV的时候，会首先在呢次中读取，若没有命中，则从level i的文件读取，若没有命中则从level i + 1层的文件读取（这里 i 依次取值0,1,...,N)。最终，要么命中，要么数据不存在。 这里有个问题，若第i层某个文件F多次被读入内存，却没有命中欲读取的KV，而是从其更高level的文件中读取到了KV。 系统为每个文件维护一个计数器（拥有一个初始值），当这个文件出现上述事件后，该文件的计数器就减1。当计数器减少到0了，就会触发对该文件的Compaction操作，称之为Seek Compaction。

**allowed_seeks计数器初始值**

在leveldb中，每个文件维护一个allowed_seek的计数器，该计数器的初始值在文件创建之初就会分配一个值。这个值是怎么分配的呢？

在代码中有如下一段注释：
```
      // We arrange to automatically compact this file after
      // a certain number of seeks.  Let's assume:
      //   (1) One seek costs 10ms
      //   (2) Writing or reading 1MB costs 10ms (100MB/s)
      //   (3) A compaction of 1MB does 25MB of IO:
      //     1MB read from this level
      //     10-12MB read from next level (boundaries may be misaligned)
      //     10-12MB written to next level
      // This implies that 25 seeks cost the same as the compaction
      // of 1MB of data.  I.e., one seek costs approximately the
      // same as the compaction of 40KB of data.  We are a little
      // conservative and allow approximately one seek for every 16KB
      // of data before triggering a compaction.
```

简单的估算出，一次文件的Seek操作相当于Compact 16KB的文件数据。那么，给定一个文件，对该文件作一次Compaction操作，相当 文件大小(单位KB) / 16KB。 这个值就作为allow_seeks计数器的初始值。

当某个文件的allowed_seeks计数器减少到0的时候， 说明这个文件经过了 allowed_seeks初始值次Seek 操作，并且没有命中KV，而从更高level中命中了KV，这就造成了读放大。在这种情况下，需要通过一次Compaction操作干掉这个文件，达到减少读放大的目的。

** 检查触发Compaction**
在leveldb中，每次写操作前，和读操作后，会有机会检查是否触发Compaction操作。

## 3.5 Compaction操作流程

这里不展开Manual Compaction。无论Size Compaction还是Seek Compaction，其基本流程是相同的（下文不具体区分两种类型的Compaction, 均用Compaction操作代替）。 具体流程如下：

* 1) 从level i 中选取一个文件或多个文件，（若 i = 0, 会选取多个文件）；
* 2) 再从level i + 1中选取若干文件，这些文件的Key 范围完全包含level i 中选取的文件的Key范围；
* 3) 将这2组文件同时读入内存，按Key字典序排序，相同的Key按Seq由大到小排序；
* 4) 依次处理3)中产生的KV序列，按照一定规则，去除无效数据，保留下来的KV依然有序；
* 5) 将4)中的KV序列按照一定格式写入磁盘文件；
* 6) 将5)中产生的文件添加到level i + 1层中，删除作为Compaction 输入的文件；
* 7) 将本次文件组织结构的改变(删除，增加文件)，记录到日志，同时新建Version用于表示当前文件组织结构。

上述流程如下图表示：

## 3.6 选取参与Compaction操作的文件

在leveldb目前实现中，除了level 0，其他level上的Compaction操作每次值选一个文件作为输入。对于level 0而言，由于文件之间Key范围有重叠，所以，对于某一个Key范围而言，可能包含多个文件。因此，在Compaction的过程中，需要将这些文件全部包含。

在leveldb中，Size Compaction的优先级要高于Seek Compaction。这样安排显然也是合理的。

Compaction操作先从level i 上选取输入文件，根据level i 中选取输入的文件来确定level i + 1上的文件。 因此，我先来叙述如何从level i上选取输入文件，对于Size Compaction和Seek Compation而言，选取的依据有所不同。 但对level i + 1上选取的文件方式则是相同的。

**Size Compaction 在level i 层选取文件**

在每一次文件组织结构变化时，leveldb会为每一个level计算一个分数(Score), 除level 0, Score计算公式是：```score = TotalFileSize / MaxFileSize ```，也就是说，将该level上总的数据量数值 除以 该level上最多能容纳的数据量数值。 系统中有Ｎ个level, 记录具有最大值的Score以及对应的level。

这里需要注意的是，level 0上的score计算方式有些不同， 由于level 0上的文件之间非有序的。为了优化读，不希望level 0有过多文件。 当文件数量超过一定阈值就触发compaction。 因此， level 0上的score计算公式为: Level 0 上文件总数量 除以 level 0 触发Compaction文件数的阈值。

当触发Compaction的时候，会优先检查这个Score值。若score >= 1, 需要对该level上选取一个文件进行Compaction操作。

另外，对于具体level而言，包含很多文件，系统会记录最近一次发生在该level上的compaction的起始点。下一次，从这个点开始选取新的文件参与compaction操作。

**Seek Compaction 在level i层选取文件**

在每一次读操作中，有机会(不是每一次)更新文件的allowed_seeks计数器，当这个计数器值为0后，将该文件记录下。当下次触发Compaction操作后，会将该文件作为Compaction的输入。 这里需要注意的是，若多个文件都同时到达Seek Compaction条件，系统只记录从最近一次Compaction开始第一个到达条件的文件。

**Level i + 1层上选取文件**

从level i + 1层上选取文件的依据是level i层上已选取的文件。首先，对于level i上选取的文件，找出其Key的范围，用[smallest, largest]来表示。 根据这个Key范围，从level i + 1上找到完全包含该范围的最小文件集。

leveldb在这里做了优化，在上述选取的level i + 1文件数量不变的情况下，试图扩大level i 中参与Compaction过程的文件数量。具体的做法是：
* 1) 计算出level i + 1上选取文件集合的Key范围；
* 2) 根据1) 计算出的文件范围，确定level i 上包含该范围的最小文件集合；
* 3) 计算出2)中选取的文件集合的总数据量；
* 4) 在满足扩展level i 层参与compaction操作的条件时，level i 层参与compaction操作的文件数量增加。 这些条件是： a. level i选取的文件数量增加; b. 且总的参与compaction的数量量不超过阈值; c. 且没有扩大level i + 1层的文件数量。

为了优化Compaction过程，在选取level + 1层上参与compaction文件的时候，会根据其Key范围，选取level + 2层（如果存在的话）上包含该范围的最小文件集合。后文叙述为什么这样作。

## 3.7 Compaction操作的详解

在选取了compaction操作所需的文件后，就要进行具体去重冗余数据的过程了。 这些文件会被读入内存，在内存中，这些数据按Key有序（按用户自定义或者系统默认的Comparator确定Key的先后顺序），相同的Key按其Sequence由大到小排序。由于文件中的KV是有序的，这个过程主要消耗在磁盘IO。

进过上述处理后，相同Key的数据会连续的排列在一起，并且Sequence较大的靠前。循环处理上述每一个KV. 这里主要是判断这个KV是否删除掉，若不删除就写入到输出文件中。

问了方便描述，假设系统中包含任何snapshot记录。那么删除的依据是：
* 1) 对于相同Key的KV集合（按Sequence降序）而言，第一个Keyr若是DELETE操作，判断在更高level上是否存在该K， 若不存在, 则删除该KV， 否则保留该KV。无论第一个KV是删除还是保留，从该Key序列的第2个KV开始，都删除掉。
* 2) 若对于相同Key的KV集合而言，第一个Keyr若是PUT操作，则保留该KV，并且删除后续相同的Key的KV。

被保留的KV需要按处理的顺序（显然也是有序的）写入输出文件。

**解释删除KV依据**

对于相同Key的KV集合，排在最前面的KV具有最大的Sequence，表明该KV是最近写入的。 

若该KV是DELETE操作，如果更高level不包含该KEY，那么该KV集合就可以全部删除掉了。 若更高level存在该KV, 那不能直接删除该KV了，否则，读操作会读到脏数据。若不存在，只需要保留第一个KV，该KV集合中其他的数据就可以完全删除掉了。

显然这里可以**进一步优化**，若更高层KV也是DELETE操作，那就可以安全删除这个KV集合了。

若该KV操作是PUT操作，那么，PUT操作是可以覆盖掉该KV集合后续所有操作的。所以，该集合中从第2个KV开始都可以删除掉了，无论是什么类型操作。


以上是在假设系统中不包含任何snapshot记录情况下的删除数据依据。接下来，分析在系统保护snapshot情况下是如何去除冗余数据的。
若系统中存在SNAPSHOT，在删除数据的时候，需要考虑改KV是否对最早的SNAPSHOT可见。若可见，则不能删除，直到该SNAPSHOT被释放后才可以删除。如何判断某KV是否对SNAPSHOT Si可见？
SNAPSHOT的sequence 为Si, 相同Key的KV集合某个Key X（记其sequence Sx)。 若该KV集合中第一次出现KV满足Sx <= Si，那么该KV，就记为对SNAPSHOT Si可见KV。 某相同Key构成的KV集合，存在对最早SNAPSHOT Si可见的KV X， 那么，大于等于KV X的sequence的KV都需要保留，不能删除。而对于，该KV集合的剩余部分构成集合，可以按上述删除原则来判定是删除还是保留。 这里不重复叙述了。

保留的KV被顺序写入输出文件中。

#### 3.8 Compaction操作输出文件

对于Compaction过程中产生的新文件，将将添加到level i + 1层中，同时需要将参与Compaction的输入文件从level i 和level i + 1中删除掉。这个过程改变了当前文件的组织结构。正如2.1节所述，文件的组织结构变动了，系统需要创建一个新的文件结构视图来替换当前的视图。

而文件变动的具体信息保存在VersionEdit中。具体过程如下：

* 1) 系统根据当前文件结构描述Current Version的内容，结合本次VersionEdit, 创建新的Version;
* 2) 对于新的文件结构（Version中记录）中的每一个文件，为每一层计算一个Score, 用于反映是否触发compaction操作，最后选择一个数值最大的Score以及对应的level；
* 3) 对于新增加的文件计算allowed_seek计数器的初始值；
* 4) 将本次VersionEdit对象序列化后，记录到MANIFEST文件中；
* 5) 将新的Version替换系统当前使用的Version。

#### 3.9 小结

本章节主要叙述了Leveldb的Compaction过程的实现。

--BY 丰茂(@丰茂IO)
