# LSM 树

## Log-Structured-Merge Tree

- 一种存储结构，目前 HBase, LevelDB, RocksDB 这些 NoSQL 存储都是采用的 LSM 树

## 核心特点

- LSM 树的核心特点是利用**顺序写**来提高**写性能**，但因为**分层** (此处分层是指的分为内存和文件两部分) 的设计会**稍微降低读性能**
- 通过**牺牲小部分读性能换来高性能写**，使得 LSM 树成为非常流行的存储结构

![LSM](images/LSM.drawio.svg)

如上图所示，LSM 树有以下**三个重要组成**部分：

### 1 MemTable

- MemTable 是在**内存**中的数据结构，用于保存**最近更新的数据**，会按照 Key **有序**地组织这些数据
- LSM 树对于具体如何组织有序地组织数据并没有明确的数据结构定义，例如 Hbase 使跳跃表来保证内存中 key 的有序。
- 因为数据暂时保存在内存中，内存并不是可靠存储，如果断电会丢失数据，因此通常会通过 WAL (Write-ahead logging，预写式日志) 的方式来保证数据的可靠性。

### 2 Immutable MemTable

- 当 **MemTable 达到一定大小后**，会转化成 Immutable MemTable
- Immutable MemTable 是将转 MemTable 变为 SSTable 的一种**中间状态**
- 写操作由**新的 MemTable 处理**，在转存过程中**不阻塞**数据更新操作

### 3 SSTable (Sorted String Table)

![sstable](images/SSTable.drawio.svg)

- **有序键值对集合**，是 LSM 树组在磁盘中的数据结构
- 为了加快 SSTable 的读取，可以通过**建立 key 的索引**以及 **布隆过滤器** 来加快 key 的**查找**
- 更新操作
  - LSM 树会将所有的数据插入、修改、删除等**操作记录**保存在**内存**之中，当此类操作**达到一定的数据量**后，再**批量**地**顺序**写入到**磁盘**当中
  - 这与 B+ 树不同，B+ 树数据的更新会直接在原数据所在处修改对应的值，但是 LSM 数的数据更新是**日志式**的，当一条数据更新是直接 **append** 一条更新记录完成的
  - 这样设计的目的就是为了顺序写，不断地将 Immutable MemTable flush 到持久化存储即可，而不用去修改之前的 SSTable 中的 key，保证了顺序写。
- 因此当 MemTable 达到一定大小 flush 到持久化存储变成 SSTable 后，在不同的 SSTable 中，可能存在相同 Key 的记录，当然最新的那条记录才是准确的。这样设计的虽然大大提高了写性能，但同时也会带来一些问题：
     1. **冗余存储**，对于某个 key，实际上除了最新的那条记录外，其他的记录都是冗余无用的，但是仍然占用了存储空间。因此需要进行 Compact 操作 (**合并多个 SSTable**) 来清除冗余的记录。
     2. 读取时需要从最新的**倒着查询**，直到找到某个 key 的记录。最坏情况需要查询完所有的 SSTable，这里可以通过前面提到的**索引** / **布隆过滤器**来**优化查找速度**。

## LSM 树的 Compact 策略

从上面可以看出，Compact 操作是十分关键的操作，否则 SSTable 数量会不断膨胀。在 Compact 策略上，主要介绍两种基本策略：size-tiered 和 leveled。

> 不过在介绍这两种策略之前，先介绍三个比较重要的概念，事实上不同的策略就是围绕这三个概念之间做出权衡和取舍。

### 三个重要概念

#### 1 读放大

读取数据时实际读取的数据量大于真正的数据量

> 例如在 LSM 树中需要先在 MemTable 查看当前 key 是否存在，不存在继续从 SSTable 中寻找。

#### 2 写放大

写入数据时实际写入的数据量大于真正的数据量

> 例如在 LSM 树中写入时可能触发 Compact 操作，导致实际写入的数据量远大于该 key 的数据量。

#### 3 空间放大

数据实际占用的磁盘空间比数据的真正大小更多

>上面提到的冗余存储，对于一个 key 来说，只有最新的那条记录是有效的，而之前的记录都是可以被清理回收的。

### Compact 策略

#### Size-tiered 策略

![lsm_size_tried.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630721607000/3b2c820cc0474e238f3895bd1bcf070d.png)

- Size-tiered 策略**保证每层 SSTable 的大小相近**，同时**限制每一层 SSTable 的数量**
- 如上图，每层限制 SSTable 为 N，当每层 SSTable 达到 N 后，则触发 Compact 操作合并这些 SSTable，并将合并后的结果写入到下一层成为一个更大的 sstable。
- 由此可以看出，当层数达到一定数量时，**最底层的单个 SSTable 的大小会变得非常大**
- 并且 size-tiered 策略会导致**空间放大比较严重**。即使对于同一层的 SSTable，每个 key 的记录是可能存在多份的，只有当该层的 SSTable 执行 compact 操作才会消除这些 key 的冗余记录。

#### Leveled 策略

![lsm_level_compact.jpg](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630721607000/0e0dc748be9e44afb32cada73a6b729e.jpg)

- Leveled 策略也是**采用分层**的思想，**每一层限制总文件的大小**
- 但是跟 size-tiered 策略不同的是，leveled 会**将每一层切分成多个大小相近的 SSTable**
- 这些 SSTable 是这一层是 **全局有序的**，意味着一个 key 在**每一层至多只有 1 条记录**，**不存在冗余记录**
- 之所以可以保证全局有序，是因为合并策略和 size-tiered 不同，接下来会详细提到。

![lsm_sstable.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630721607000/d31f001cb1524a55a91c4c0ee7869866.png)

##### 合并过程

假设存在以下这样的场景：

1. L1 的总大小超过 L1 本身大小限制
    - ![lsm_compact_01.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630721607000/1175e99f05134dc0b18fe1fc8187a39f.png)
2. 此时会从 L1 中**选择至少一个文件**，然后把它跟**L2 有交集的部分** (非常关键) 进行合并。生成的文件会放在 L2
   - ![lsm_compact_02.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630721607000/63907b21f4fb46e79b22ebe0a233138c.png)
   - 如上图所示，此时 L1 第二 SSTable 的 **key 的范围覆盖了 L2 中前三个 SSTable**，那么就需要将 L1 中第二个 SSTable 与 L2 中前三个 SSTable 执行 Compact 操作。
3. 如果 L2 合并后的结果**仍旧超出** L2 的阈值大小，需要**重复**之前的操作，选至少一个文件然后把它合并到下一层:
   - ![lsm_compact_03.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630721607000/801f1beb92204f56ad7081eba7199688.png)
4. ![lsm_compact_04.png](https://fynotefile.oss-cn-zhangjiakou.aliyuncs.com/fynote/908/1630721607000/56d4defefa1548ccbf833c34c4865b25.png)

- Leveled 策略相较于 size-tiered 策略来说，**每层内 key 是不会重复的**
- 即使是**最坏的情况**，除开最底层外，其余层都是重复 key，按照相邻层大小比例为 10 来算，**冗余占比也很小**
- 因此**空间放大问题得到缓解**。但是**写放大问题会更加突出**。举一个最坏场景，如果 LevelN 层某个 SSTable 的 key 的范围跨度非常大，覆盖了 LevelN+1 层所有 key 的范围，那么进行 Compact 时将涉及 LevelN+1层的全部数据。

## 总结

- LSM 树是非常值得了解的知识，理解了 LSM 树可以很自然地理解 Hbase，LevelDb 等存储组件的架构设计
- ClickHouse 中的 MergeTree 也是 LSM 树的思想，Log-Structured 还可以联想到 Kafka 的存储方式。
- 虽然介绍了上面两种策略，但是各个存储都在自己的 Compact 策略上面做了很多特定的优化，例如 Hbase 分为 Major 和 Minor 两种 Compact，这里不再做过多介绍，推荐阅读 RocksDb 合并策略介绍。
