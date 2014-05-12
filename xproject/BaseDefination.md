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

**日志文件实现**

有关日志实现的代码文件为db/log_format.h log_reader.{h,cc}, log_writer.{h,cc}.
其中log_format.h 定义了Record的type;
log_writer.{h, cc}实现了写日志的逻辑。 其中，AddRecord是对外开放的接口。
```
Status Writer::AddRecord(const Slice& slice) {
  const char* ptr = slice.data();
  size_t left = slice.size();

  // Fragment the record if necessary and emit it.  Note that if slice
  // is empty, we still want to iterate once to emit a single
  // zero-length record
  Status s;
  bool begin = true;
  do {
    const int leftover = kBlockSize - block_offset_;
    assert(leftover >= 0); 
    if (leftover < kHeaderSize) {
      // 如果剩余空间小于Header的大小，就填充0；
      // 重新起一个新的Block;
      if (leftover > 0) {
        // Fill the trailer (literal below relies on kHeaderSize being 7)
        assert(kHeaderSize == 7);
        dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover));
      }
      block_offset_ = 0;
    }

    // Invariant: we never leave < kHeaderSize bytes in a block.
    assert(kBlockSize - block_offset_ - kHeaderSize >= 0);

    const size_t avail = kBlockSize - block_offset_ - kHeaderSize;
    const size_t fragment_length = (left < avail) ? left : avail;
    }
    // 判断每一个Record的Type
    const bool end = (left == fragment_length);
    if (begin && end) {
      type = kFullType;
    } else if (begin) {
      type = kFirstType;
    } else if (end) {
      type = kLastType;
    } else {
      type = kMiddleType;
    }

    // 写入Record
    s = EmitPhysicalRecord(type, ptr, fragment_length);
    ptr += fragment_length;
    left -= fragment_length;
    begin = false;
    // 若数据没有写完，这些数据会写在连续的Record中；
  } while (s.ok() && left > 0);
  return s;
}
```

将数据写入文件的函数代码如下：
```
Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr, size_t n) {
  assert(n <= 0xffff);  // Must fit in two bytes
  assert(block_offset_ + kHeaderSize + n <= kBlockSize);

  // 填充Header
  // Format the header
  char buf[kHeaderSize];
  buf[4] = static_cast<char>(n & 0xff);
  buf[5] = static_cast<char>(n >> 8);
  buf[6] = static_cast<char>(t);

  // Compute the crc of the record type and the payload.
  uint32_t crc = crc32c::Extend(type_crc_[t], ptr, n);
  crc = crc32c::Mask(crc);                 // Adjust for storage
  EncodeFixed32(buf, crc);

  // 将数据写入顺序文件
  // Write the header and the payload
  Status s = dest_->Append(Slice(buf, kHeaderSize));
  if (s.ok()) {
    s = dest_->Append(Slice(ptr, n));
    if (s.ok()) {
      s = dest_->Flush();
    }
  }
  block_offset_ += kHeaderSize + n;
  return s;
}
```

## 2.2 磁盘文件（SST）的格式

**磁盘文件布局**

在Leveldb中，数据最终以SST文件形式存储在磁盘中的。在doc/table_format.txt文件中有记录该文件的一些信息。SST文件以4K大小的块(Block)构成。这些Block按其存储的数据可以分为 数据块(Data Block), 元数据(Meta Data Block), 索引数据块(Index Block)。在文件的结尾存成一个固定长度的Footer, 包含元数据索引句柄，数据索引句柄，或许有填充数据，Magic Code。图2-2展示了SST文件的整体布局。

![](http://i.imgur.com/UVLVK2V.png)

图2-2 SST Layout

**Data Block**

在SST文件中，目前有2中类型的Block, 一种是数据Block， 存储用户数据，如图2-2中的Data Block Layout所示；另外一种是Meta Data Block。两种Block的区别主要在于索引的不同。对于Data Block而言，并不是每一个Data Entry都会创建索引，而是隔一定数量的Data Entry，才对这个Entry创建索引。索引是一个Map, 其Key就是被索引Data Entry的key, 其Value是这个Data Entry在文件中的Offset. 这个Map就以数组的形式存储在Block结尾处。Map中的每一项内容称之为RestartPoint。 由于为了节约存储空间，Data Block中的Data Entry采用前缀压缩的方式存储。具体而言，若第i+1个DataEntry 与 第i个Data Entry的Key相似部分的话，第i + 1个Data Entry不需要存储相同的内容，只存储不同部分的内容。但是，被索引的Data Entry需要存储完整的Key。考虑到Key是有序的，相邻Key拥有相同内容是比较常见的。这种组织方式能够节约存储空间。

这种存储格式节约存储空间，又建立索引加快查找。有了数组型的索引，并且Key是有序的，可以二分查找法快速定位到目标Key的前一个索引Key, 然后通过线下查找定位具体的Key，图2-3展示了如何查找一个给定的Key。

![](http://i.imgur.com/OYAT4sa.png)

如图2-3所示，上部分为DataEntry的存储区域，下部分为索引存储区。蓝色部分为创建索引的Data Entry，红色为目标Data Entry。首先，根据索引区域，二分查找定位到目标Key(红色）的前一个索引点，然后采用二分查找定位到红色Key。 在SST文件中，这种类型的Block存储数据，Block级的索引块。

**Meta Data Block**

对于元数据块（Meta Data Block）而言，建立索引的方式有明显不不同，是对Block中的每一个Data Entry都建立索引。这种Block我们可以理解为，采用自然数0,1,2 .. 作为Key，并不存储在Block中（逻辑上的Key), 每个Entry的均建有索引，存储在Block的结尾处。索引数据是一个数组，数组元素是对应的Data Entry在文件中的Offset. 若要查找第i个Data Entry， 通过索引（数组下标）中的数据，得到具体的Data Entry。相对Data Block而言，这种索引查找效率高，占用更多的存储空间，适合存储元数据。

**Meta Index Block**

在SST文件中，接近这数据Block存放的是Meta Data Block, 而文件可能存在多个Meta Data Block, 为了能够快速检索出具体的Meta Data Block, SST文件对Meta Data Block作了索引。索引数据存储在一个Data Block中，其Key是Meta Data Block的名称，其Value是一个Block句柄，该句柄记录被索引的Block在文件中的Offset 和 Size。根据这个句柄，能够读出具体Block的数据。

在目前的实现中，整个文件只有一个Meta Data Block, 所以Meta Index Block中只有一条记录。

**Index Block**

SST文件为每一个Block创建了索引。这些索引数据存储为一个Data Block, 位于Meta Index Block 结尾处。索引数据组织成一个Key-Value 表，其Key是一个不小于Data Block中的最大Key(显然是Data Block的最后一个Key)，并且小于相邻的下一个Block的最小Key 。为了节省空间，在创建SST文件的时候，会在前一个文件的最大Key 与 相邻文件的最小Key之间选择一个“最短”的Key, 这样能够节省存储空间。索引表的Value是一个Block句柄。

通常，将SST的Index Block 缓存在Cache中，这样能够加快Search Key的过程。

**Footer**
在SST文件的结尾，存储了一段特殊的，固定长度的数据，称之为Footer。这段固定长度的数据中，依次记录了Meta Index Block的句柄，Index Block的句柄，填充数据（如果需要的话），
