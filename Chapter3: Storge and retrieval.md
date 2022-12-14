# 3.1 Data structure that power your database
## 3.1.1 Hash indexes

**基本结构(p72-p75)：**

1. 内存：hashmap
2. 磁盘：segments and compaction
3. 其他实现细节
   - File format: binary format. bytes length of string + raw string
   - delete: add a special record called tombstone, for deleting signal in compaction
   - crash recovery: snapshot of memory hash map
   - partially written records: use checksum to find it
   - concurrency control: only one writer thread running

**为什么使用append-only而不是update的方式存储？(p75)**

1. 顺序写入比随机写快，尤其在HDD上。  即使SSD顺序写叶更快，原因见：page83
2. crash恢复更方便。 如果是
3. compaction避免了文件碎片化

**hash table index的局限(p75)：**

1. 依赖内存大小。  如果key数量极大，就不太适用。  基于硬盘实现hashmap，非常复杂，成本较高
2. range query查询低效

## 3.1.2 SSTables（Sorted string table） and LSM-Trees

### 基本结构(p76)
在hash index disk segment基础上，要求segment内按照key排序

### 优势（p76-p77）
1. compaction: 归并排序即可。同key但value不同的，丢弃更早的value
2. 内存索引: 稀疏索引即可，提供了被检索key的segment offset
3. 压缩：稀疏索引对应的block，可以压缩存储，节省磁盘I/O和space
4. 支持高效range query，由于disk write是顺序的，可支持很高的写吞吐

### 如何构建sorted index？
1. 数据结构：内存中维护一个平衡树（如红黑树），有序。 一般叫做**memtable**
2. spill：内存树到一定阈值，spill到磁盘segment
3. read：优先查内存。查不到的查磁盘segment，由近及早
4. compaction：定期后台合并，丢弃update的value、删除的key

### 劣势
crash recovery，内存直接丢了。 解法：write-ahead， write之前，先打印log（类似binlog），crash后从log恢复

### LSM-Tree历史沿革(p78-p79):
1. 上述算法，Patrick O’Neil等人先使用Log-Structured Merge-Tree（LSM-Tree）来描述；
2. **SSTable**和**memtable** 名次来自于Google Bigtable的论文；
3. LevelDB，RocksDB使用的KV引擎收到Bigtable启发；
4. Elasticsearch 和Solr 中的Lucene引擎，也使用了类似的idea，来构建key(item) -> value(document id list) index

### 性能优化
1. 查询不到的key：引入bloom filter
   >It can tell you if a key does not appear in the database
2. compaction的顺序和时机
   1. size-tiered（HBase）： 新的小的segment合入到旧的大的
   2. leveled（LevelDB and RocksDB）：（寥寥几句话，没太看懂...） TODO：可以详细研究
   Cassandra两种都支持



## B-Trees

### structure(p80-p81)
1. block/page: 对应图中一个树的每一个节点。size是固定的，类似disk底层的管理方式。  不像log-structure index, 是可变大小。 每个page中可容纳的reference数量，叫做 _branching factor_，一般大小是几百
2. 存储在disk上，而不是内存
3. update: 从root开始，顺着树查询
4. insert: 查询，然后在合适的位置插入。  如果当前page超出 _branching factor_，需要分裂为两个page，更新父亲节点。  有可能递归向上传播。 这样可以保证树的平衡，可以在O(logn)查询效率
5. level：一般3、4层就够
   >A four-level tree of 4 KB pages with a branching factor of 500 can store up to 256 TB
   
![](/images/b-tree_structure.png)

### reliability(p82)
1. 现象：B-tree的update操作是直接disk原地更改，包括insert时候可能触发的page split
2. crash recovery: a write-ahead log
3. concurrency control: latches (一种轻量级锁)

### optimization (p82-p83)
1. copy-on-write: write的时候copy换一个location写入，parent page用版本管理，新版本reference指向新的location。 版本管理也可以用来做concurrent control
2. key abbr for saving space. no specific info
3. B+tree: pointer to siblings page from left to right in leaf level


## Compare B-Trees and LSM_Trees

### LSM-tree的优势
1. lower write amplification: 因此而拥有更好的写吞吐能力
   >the more that a storage engine writes to disk, the fewer writes per second it can handle within the available disk bandwidth.
2. reduced fragmentation: 顺序写入，不像B-tree那么碎片化；更容易压缩；

### LSM-tree的劣势
1. >compaction process can sometimes interfere with the performance of ongoing reads and writes.
2. LSM-tree中，同一个key可能存在于多个segment中；B-tree相反，都在一个page里，更好做读写隔离（如事务的隔离）


## Other Indexing Structures

### 在index中存储value？
1. clustered index (storing all row data within the index): 需要处理index和数据、多个index之间的一致性问题；duplicate导致额外的存储消耗；write需要写多个地方
2. nonclustered index (storing only references to the data within the index)：read有性能损失

### 多列索引
1. concatenated index：就是把多个key拼起来而已（类似doris的前缀索引），只能按照拼的顺序做前缀筛选，不能单独筛选非一级索引
2. Multi-dimensional index：支持多列同时，不同条件的筛选，真正意义上的多列索引。 一般可以用R-tree实现

### Full-text search and fuzzy indexes
常用于搜索引擎，支持纠错（一定的编辑距离）、同义词等，属于信息检索的范畴。  一般用到 _trie_ 等数据结构



### Keeping everything in memory
1. 完全使用内存，然后使用snapshot，分布式，带电池的RAM，NVM（断电后仍可保存数据的内存新型硬件）等方式提供reliability
2. 反直觉地，in-memory database快并不是因为不用从disk读。  内存足够大的话，OS会把最近用的cache到memory。 主要的“快”来自于不用把内存里的数据合适方式编码后存储到disk。  因此，in-memory还有一个优点，是可以提供更复杂的数据结构（如redis）
3. 另外，即便使用类似OS的方式，把很久没用到的spill到disk，再把最近用到的加载到内存。  但是相比OS是以page的粒度，in-memory db可以以record的粒度来做，粒度更精细


# Transaction Processing or Analytics?
这节没讲的内容都太熟了，没啥可以记的，就放个比较olap和oltp的图吧
![](/images/diff-olap-oltp.png)

## Column-Oriented Storage

1. 列存：想法很简单，就是把每一列的相近数据存在一起，列间拆分。而不是把相近的行存在一起。
2. 试用场景：analytics/olap, 查询中scan的列较少，但是scan大量的row。 如果行存，会导致药scan几乎所有的数据进内存

![](/images/relational2column.png)

如图，列存中有一个关键点，每列中包含的行顺序要一致

### Column Compression

**bitmap encoding** 是一种有效的压缩方法

1. column的distinct value肯定少于row的disitnct
2. 用n个不同的数代表column所有的distinct value（类似hashmap）。  bitmap的每个bit位代表一个row，顺序也和row对应
3. 当数据量很大的时候，bitmap中的1会很稀疏。可以使用run-length encoding解决
4. 用bitmap之间位运算来实现OR、AND where conditions。得益于每个bitmap的二进制位数相同（行数相同）；且即使column不同，行的顺序也相同
```sql
   WHERE product_sk IN (30, 68, 69)
   -- Load the three bitmaps for product_sk = 30, 68, 69
   -- calculate the bitwise OR of the three bitmaps
   -- then，那些bit位为1的，就是要查询的行。可以借此拿到行对应的索引（顺序相同）

   WHERE product_sk = 31 AND store_sk = 3
   -- Load the bitmaps for product_sk = 31, store_sk = 3
   -- calculate the bitwise AND
   -- 符合这个条件的行，行内product_sk=31和store_sk=3同时成立，即两个bitmap的同一bit位（对应行），值都是1
```

详细见图：
![](/images/column-compression.png)

**memory bandwidth and vectorized processing**
这块书中描述比较简略，可以参考书中引用深入

### Sort Order in Column Storage
1. column元素如果要重排序（即在原本append插入顺序的基础上，重排序），需要同一行一起排，否则将找不到属于同一行的column
2. DBA可以参照日常query分布来决定按照哪个column来排序(如date)。也可以分一级、二级sort key，是级联的
3. 优点：加速查询（前缀索引的效果）；排序的列更容易压缩，因为相同的值连续排列
4. 多种排序共存：可以在分布式replication存储上，使用不同的排序方式。然后在查询的时候，挑选最适合当前query的排序。 和row-oriented不同的地方在于，column-orinted实实在在存了多次，而row-oriented可以只存不同的索引，原始数据只有一份

### Writing to Column-Oriented Storage

基于以上对column-oriented的描述，怎么write?

1. 类似LSM-tree，先在内存里积累，然后和disk上的数据merge，保持disk上的列存、有序
2. 查询需要结合memory和disk上的数据

### Aggregation: Data Cubes and Materialized Views
1. 物化视图：轻度聚合，单独存储。 不想OLTP db里的视图，只是一层虚拟视图，查询视图的sql会转为查询table的sql，做不到加速查询，只是抽象对外的接口
2. 优势：加速SUM等特定的聚合查询
3. 劣势：写成本较大（但是对于“重读”的数仓，用空间来换时间还是划算的）；对应需要从最细粒度原始数据的查询无能为力，比如查询分位值等

