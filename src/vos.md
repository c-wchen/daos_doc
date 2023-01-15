# 版本对象存储（VOS）

版本化对象存储(VOS)负责提供和维护持久对象存储，该存储支持字节粒度访问和[DAOS池](https://github.com/daos-stack/daos/blob/master/docs/storage_model.md#DAOS_Pool)中单个分片(shard)的版本控制。它在持久性内存中维护其元数据，根据可用存储和性能要求，数据可以存储在持久性内存中，也可以存储在块存储（block storage）中。它必须以最小的开销提供该功能，以便在延迟和带宽方面的性能尽可能接近底层硬件的理论性能。它在持久和非持久内存中的内部数据结构也必须支持最高级别的并发，以便在现代处理器体系结构的核心上扩展吞吐量。最后，至关重要的是，它必须验证所有持久化对象数据的完整性，以消除静默数据损坏的可能性，无论是在正常操作中还是在所有可能的可恢复故障下。

本节提供实现上述为DAOS构建版本化对象存储时讨论的设计目标的详细信息。

本文档包含以下内容:

- Persistent Memory based Storage
  - [In-Memory Storage](https://github.com/daos-stack/daos/tree/master/src/vos#63)
  - [Lightweight I/O Stack: PMDK Libraries](https://github.com/daos-stack/daos/tree/master/src/vos#64)
- VOS Concepts
  - [VOS Indexes](https://github.com/daos-stack/daos/tree/master/src/vos#711)
  - [Object Listing](https://github.com/daos-stack/daos/tree/master/src/vos#712)
- Key Value Stores
  - [Operations Supported with Key Value Store](https://github.com/daos-stack/daos/tree/master/src/vos#721)
  - [Key in VOS KV Stores](https://github.com/daos-stack/daos/tree/master/src/vos#723)
  - [Internal Data Structures](https://github.com/daos-stack/daos/tree/master/src/vos#724)
- [Key Array Stores](https://github.com/daos-stack/daos/tree/master/src/vos#73)
- Conditional Update and MVCC
  - [VOS Timestamp Cache](https://github.com/daos-stack/daos/tree/master/src/vos#821)
  - [Read Timestamps](https://github.com/daos-stack/daos/tree/master/src/vos#822)
  - [Write Timestamps](https://github.com/daos-stack/daos/tree/master/src/vos#823)
  - [MVCC Rules](https://github.com/daos-stack/daos/tree/master/src/vos#824)
  - [Punch Propagation](https://github.com/daos-stack/daos/tree/master/src/vos#825)
- Epoch Based Operations
  - [VOS Discard](https://github.com/daos-stack/daos/tree/master/src/vos#741)
  - [VOS Aggregate](https://github.com/daos-stack/daos/tree/master/src/vos#742)
- [VOS Checksum Management](https://github.com/daos-stack/daos/tree/master/src/vos#79)
- [Metadata Overhead](https://github.com/daos-stack/daos/tree/master/src/vos#80)
- Replica Consistency
  - [DAOS Two-phase Commit](https://github.com/daos-stack/daos/tree/master/src/vos#811)
  - [DTX Leader Election](https://github.com/daos-stack/daos/tree/master/src/vos#812)



## 基于持久性内存的存储

### In-Memory Storage

VOS被设计为使用持久化内存存储模型，利用新的NVRAM技术可能实现的字节粒度、亚微秒级存储访问。与传统的存储系统相比，应用程序和系统元数据以及小的、碎片化的和不对齐的I/O的性能发生了颠覆性的变化。直接访问字节可寻址的低延迟存储开辟了新的领域，元数据可以在不到一秒的时间内扫描，而无需费心查找时间和对齐。

VOS依赖于基于日志的体系结构，使用持久性内存主要维护内部持久性元数据索引。实际数据可以直接存储在持久内存中，也可以存储在基于块的NVMe存储中。DAOS服务有两层存储:存储类内存(SCM)用于字节粒度的应用程序数据和元数据，NVMe用于批量应用程序数据。与目前使用PMDK方便访问SCM类似，存储性能开发工具包([SPDK](https://spdk.io/))提供对NVMe ssd的无缝高效访问。当前的DAOS存储模型涉及每个内核三个DAOS Server xstream，以及映射到NVMe SSD设备的每个内核一个主DAOS服务器xstream。DAOS存储分配可以通过使用PMDK pmemobj池在SCM上进行，也可以通过使用SPDK blob在NVMe上进行。所有本地服务器元数据将存储在SCM上的每个服务器pmemobj池中，并将包括所有当前的和相关的NVMe设备、池和xstream映射信息。请参阅[Blob I/O](https://github.com/daos-stack/daos/blob/master/src/bio/README.md) (BIO)模块以获取有关NVMe、SPDK和每服务器元数据的更多信息。在开发和修改VOS层时要特别注意，因为任何软件缺陷都可能破坏持久内存中的数据结构。因此，尽管存在硬件ECC, VOS仍然对其持久数据结构进行校验和。

VOS充分利用为支持这种编程模型而开发的[PMDK](https://github.com/daos-stack/daos/blob/master/src/vos/pmem.io)开源库，在用户空间中提供了一个轻量级的I/O堆栈。



### Lightweight I/O Stack: PMDK Libraries

虽然持久性内存可以通过直接加载/存储访问，但更新需要通过多级缓存，包括处理器L1/2/3缓存和NVRAM控制器。只有在所有这些缓存都显式刷写之后，才可以保证持久性。VOS在持久内存中维护内部数据结构，这些数据结构必须保持一定程度的一致性，以便在意外崩溃或停电后恢复操作而不会丢失持久数据。对一个请求的处理通常会导致多次内存分配和更新，这些都必须以原子方式应用。

因此，必须在持久性内存之上实现事务接口，以保证内部VOS的一致性。值得注意的是，这种事务不同于DAOS事务机制。在处理传入的请求时，持久性内存事务必须保证VOS内部数据结构的一致性，无论它们的纪元数是多少。持久性内存上的事务可以通过许多不同的方式实现，例如，撤销日志、重做日志，两者的组合，或者写时复制。

[PMDK](https://pmem.io/)是一个使用持久性内存的开源库集合，专门针对NVRAM进行了优化。其中包括libpmemobj库，它实现了称为持久性内存池的可重定位持久性堆。这包括内存分配、事务和用于持久内存编程的一般设施。事务在一个线程(不是多线程)中是本地的，并且依赖于undo日志。正确使用API可以确保在服务器发生故障后打开内存池时，所有内存操作都回滚到最后提交的状态。VOS利用这个API来确保VOS内部数据结构的一致性，即使在服务器故障的情况下。



## VOS 概念

版本化对象存储通过将VOS池(vpool)初始化为DAOS池的一个shard，提供存储目标本地的对象存储。vpool可以为多个对象地址空间(称为容器)保存对象。每个vpool在创建时都有一个唯一的UID，与DAOS池的UID不同。VOS还维护并提供了一种方法来提取统计信息，如总空间、可用空间和vpool中对象的数量。

VOS的主要目的是捕获并记录任意时间顺序的对象更新，并将这些更新集成到一个有序的、可按需高效遍历的纪元历史中。通过正确排序冲突的更新，而不需要及时序列化它们，这为并行I/O提供了重大的可伸缩性改进。例如，如果两个应用程序进程就如何解决给定更新上的冲突达成一致，它们可以独立地编写更新，并确保它们将在VOS上按正确的顺序解决。

VOS还允许丢弃与给定纪元和进程组相关的所有对象更新。此功能确保在必须放弃DAOS事务时，在为该进程组提交epoch之前，所有相关的更新都是不可见的，并成为不可变的。这确保了分布式更新是原子的——即当一个提交完成时，要么所有的更新已经被应用，要么被丢弃。

最后，VOS可以聚合对象的历史记录，以回收不可访问数据所占用的空间，并通过简化索引来加快访问速度。例如，当一个数组对象在给定的epoch内被“打孔”从0到∞时，容器一旦关闭，在此epoch之前的最新快照之后更新的所有数据都将变得不可访问。

在内部，VOS维护一个容器uuid的索引，该索引引用存储在特定池中的每个容器。容器本身包含三个索引。第一个是对象索引，用于在处理I/O请求时有效地将对象ID和epoch映射到对象元数据。另外两个索引用于维护活动和已提交的[DTX](https://github.com/daos-stack/daos/tree/master/src/vos#811)记录，以确保跨多个副本的高效更新。

DAOS支持两种类型的值，每种类型都与一个分布键(DKEY)和一个属性键(AKey)相关联:单个值和数组值。DKEY用于放置，确定使用哪个VOS池来存储数据。键标识了要存储的数据。同时指定DKEY和key的能力为应用程序提供了在DAOS中分发或共存不同值的灵活性。单个值是一个原子值，这意味着写入键时更新整个值，读取时获取最新的完整值。数组值是大小相同的记录的索引。对数组值的每次更新只影响指定的记录，并读取对请求的每个记录索引的最新更新。每个VOS池维护VOS提供了容器、对象、dkey、akey和值的每个容器层次结构，如下所示[下面](https://github.com/daos-stack/daos/tree/master/src/vos#7a)。DAOS API提供了构建在此基础接口上的通用键值和数组抽象。前者将DKEY设置为用户指定的密钥，并使用固定的密钥。后者使用数组索引的上位来创建DKEY，并使用固定的key，从而将数组索引均匀地分布到对象布局中的所有VOS池中。对于VOS描述的其余部分，Key-Value和Key-Array应使用来描述VOS布局，而不是这些简化的抽象。换句话说，它们应该描述单个VOS池中的dkey - key - value。

VOS对象不是显式创建的，而是在第一次写入时通过创建对象元数据并在所属容器的对象索引中插入对它的引用来创建的。所有对象更新都会记录每次更新的数据，这些更新可能是对象、DKEY、key、单个值、数组值punch或单个值或数组值的更新。请注意，数组对象区段的“击打”记录为零区段，而不是导致相关的数组区段或键值被丢弃。对对象、DKEY、key或单个值的击打会被记录，因此在以后的时间戳读取时不会看到任何数据。这确保了对象的完整版本历史仍然可以访问。然而，DAOS api只允许在快照中访问数据，因此VOS聚合可以积极地删除在已知快照中不再访问的对象、键和值。

[![../../docs/graph/Fig_067.png](https://github.com/daos-stack/daos/raw/master/docs/graph/Fig_067.png)](https://github.com/daos-stack/daos/blob/master/docs/graph/Fig_067.png)

在查找对象中的单个值时，遍历对象索引，以找到索引中与键匹配的历元数(近历元数)小于或等于请求的历元数的最大节点。如果找到值或负数，则返回。否则，返回一个"miss"，意味着这个键在这个VOS中从未更新过。这确保返回的是纪元历史中最近的值，而不管它们整合的时间顺序如何，并且忽略请求纪元之后的所有更新。

类似地，在读取数组对象时，遍历它的索引以创建一个聚集描述符，该描述符收集请求区间中所有纪元数小于或等于请求纪元数的对象区段片段。聚集描述符中的条目要么引用一个包含数据的区间，一个请求者可以解释为全0的穿孔区间，要么引用一个“miss”，意味着这个VOS在这个区间内没有收到任何更新。同样，这确保了对于请求范围内的所有偏移量，返回数组纪元历史中最近的数据，而不管它们的写入时间顺序如何，并且忽略请求纪元之后的所有更新。



### VOS Indexes

对象索引表的值，由OID索引，指向一个DKEY索引。DKEY索引中的值指向一个key索引。key索引中的值(由key索引)指向单个值索引或数组索引。epoch引用单个值索引，并返回在epoch时或之前插入的最新值。数组值由范围和epoch索引，并将返回该epoch可见的区段的部分。

关于对象期望的提示可以编码在对象ID中。例如，一个对象可以被复制、纠删编码、使用校验和，或者具有整数或词法dkey和/或akey。如果使用整数键或词法键，对象索引是按键排序的，这使得查询(如数组大小)更加高效。否则，键将根据索引中的散列值进行排序。128位。上面的32位用于编码对象类型和键类型，而下面的96位是用户定义的标识符，必须与容器唯一。

每个对象、dkey和key都有一个关联的化身日志。化身日志可以被描述为关联实体的创建和打孔事件的有序日志。该值路径中的每个实体都会被检查日志，以确保该实体以及该值在请求的时间是可见的。

### Object Listing

VOS提供了一个通用迭代器，可用于迭代VOS池中的容器、对象、dkey、akey、单个值和数组区段。迭代API如下面的[图](https://github.com/daos-stack/daos/tree/master/src/vos#7b)所示。

```
/**
 * Iterate VOS entries (i.e., containers, objects, dkeys, etc.) and call \a
 * cb(\a arg) for each entry.
 *
 * If \a cb returns a nonzero (either > 0 or < 0) value that is not
 * -DER_NONEXIST, this function stops the iteration and returns that nonzero
 * value from \a cb. If \a cb returns -DER_NONEXIST, this function completes
 * the iteration and returns 0. If \a cb returns 0, the iteration continues.
 *
 * \param[in]		param		iteration parameters
 * \param[in]		type		entry type of starting level
 * \param[in]		recursive	iterate in lower level recursively
 * \param[in]		anchors		array of anchors, one for each
 *					iteration level
 * \param[in]		cb		iteration callback
 * \param[in]		arg		callback argument
 *
 * \retval		0	iteration complete
 * \retval		> 0	callback return value
 * \retval		-DER_*	error (but never -DER_NONEXIST)
 */
int
vos_iterate(vos_iter_param_t *param, vos_iter_type_t type, bool recursive,
	    struct vos_iter_anchors *anchors, vos_iter_cb_t cb, void *arg);
```

泛型VOS迭代器API启用了DAOS枚举API以及支持重建、聚合和丢弃的DAOS内部特性。它足够灵活，可以遍历指定epoch范围内的所有键、单个值和区段。此外，它支持通过可见范围进行迭代。



## Key Value Stores (Single Value)

高性能模拟产生大量数据时，需要对数据进行索引和分析，以获得良好的洞察力。Key Value (KV)存储在简化复杂数据的存储和高效处理方面起着至关重要的作用。

VOS在持久性内存上提供了一个多版本、并发的KV存储，它可以动态增长，并提供快速的近历元检索和键值枚举。

虽然之前有一系列关于KV存储的工作，但大多数都专注于云环境，并没有提供有效的版本支持。一些KV存储提供了版本控制支持，但期望版本的顺序是单调递增的，而且没有近历元检索的概念。

VOS必须能够在任何时间插入KV对，并且必须能够对任何键值对象的并发更新和查找提供良好的可扩展性。KV对象还必须能够支持任何类型和大小的键和值。

### Operations Supported with Key Value Store

VOS支持四种类型的大键和大值操作。更新、查找、打孔和键枚举。

update和punch操作会将一个新键添加到KV存储中，或者记录一个已有键的新值。Punch记录特殊值“Punch”，实际上是一个负数条目，用来记录删除键时的时间。对于同一个对象、键、值或范围的更新和击打，都不允许使用相同的epoch，当试图这样做时，VOS将返回错误。

Lookup遍历KV元数据，以确定给定键在给定时期的状态。如果根本没有找到键，则返回一个"miss"，表示该键在该VOS中不存在。否则，返回小于或等于请求的epoch的近epoch或最大epoch的值。如果是特殊的“打孔”值，则意味着该键在指定的时间内被删除了。这里的值指的是内部树数据结构中的值。KV-object的键值记录作为其节点的值存储在树中。因此，在穿孔的情况下，这个值包含一个“特殊的”返回码/标志来标识穿孔操作。

VOS还支持枚举属于特定纪元的键。



### Key in VOS KV Stores

VOS KV支持从小到超大的密钥长度。对于key和dkey, VOS支持散列键或两种“直接”键:词法键或整数键。

#### Hashed Keys

最灵活的键类型是散列键。VOS对用户提供的键运行两种快速散列算法，并使用组合散列后的键值作为索引。组合散列的目的是避免键之间的冲突。为了保证正确性，仍然需要比较实际的键。

#### Direct Keys

使用散列后的键会产生无序的键。在用户的算法可能从排序中受益的情况下，这是有问题的。因此，VOS支持两种类型的键，它们不散列，而是直接解释。

##### Lexical Keys

词法键使用词法排序进行比较。这使得诸如排序字符串之类的用法成为可能。目前，词法键的长度被限制在80个字符以内。

##### Integer Keys

整数键是无符号的64位整数，因此进行比较。这使得像DAOS数组API这样的用例可以使用索引的上位作为dkey，下位作为偏移量。这使得此类对象能够使用DAOS键查询API来更有效地计算大小。

VOS中的KV存储允许用户以随机顺序维护不同KV对的版本。例如，一次更新可能发生在epoch 10，然后在epoch 5进行另一次更新，其中HCE小于5。为了提供这种级别的灵活性，KV存储中的每个键都必须保持更新/打孔的周期。索引树中元素的排序首先基于键，然后基于年代。这种排序允许相同键值的epoch落在同一子树中，从而最小化搜索成本。使用稍后描述的[DTX](https://github.com/daos-stack/daos/tree/master/src/vos#81)执行冲突解决和跟踪。DTX确保副本是一致的，失败或未提交的更新在外部不可见。



### Internal Data Structures

设计VOS KV存储需要一种树型数据结构，它可以动态增长并保持自平衡。树需要进行平衡，以确保时间复杂度不会随着树大小的增加而增加。树的数据结构有红黑树和B+树，前者是二叉查找树，后者是n元查找树。

尽管与AVL树相比，红黑树提供的平衡不那么严格，但它们通过更低的再平衡成本进行了补偿。红黑树在Linux内核、java-util库和c++标准模板库等例子中应用更为广泛。B+树与B树的不同之处在于，它们没有与内部节点相关联的数据。这可以方便在一页内存中放置更多的键。此外，B+树的叶节点是相互连接的;这意味着，与B树相比，进行一次完整的扫描只需要线性遍历所有的叶节点，这可以潜在地减少访问数据时的缓存缺失。

如前一节所述([键值存储支持的操作](https://github.com/daos-stack/daos/tree/master/src/vos#721))，为了支持更新和打卡，为每个更新或打卡请求设置一个纪元有效性范围以及相关的键，这将标记键从当前纪元到可能的最高纪元有效。更新同一个键在未来或过去的epoch上修改前一个更新或punch的结束epoch有效性。这样，对于任何给定的键-历元对查找，只有一个键的有效范围，而整个键的更新历史都会被记录下来。这有助于最近时间搜索。punch和update都有类似的键，除了一个简单的标志，用于标识被查询时间上的操作。查找必须能够在给定的时间内搜索给定的键并返回相关联的值。除了时间有效性范围之外，DAOS生成的容器句柄cookie也与树的键一起存储。如果在同一时间发生重写，则需要此cookie来识别行为。

下面的[表](https://github.com/daos-stack/daos/tree/master/src/vos#7c)中列出了一个创建KV存储的简单输入示例。基于B+树的索引和基于红黑树的索引分别显示在下面的[表](https://github.com/daos-stack/daos/tree/master/src/vos#7c)和[图](https://github.com/daos-stack/daos/tree/master/src/vos#7d)中。出于解释的目的，本例中使用了具有代表性的键和值。

**Example VOS KV Store input for Update/Punch**

| Key   | Value   | Epoch | Update (U/P) |
| ----- | ------- | ----- | ------------ |
| Key 1 | Value 1 | 1     | U            |
| Key 2 | Value 2 | 2     | U            |
| Key 3 | Value 3 | 4     | U            |
| Key 4 | Value 4 | 1     | U            |
| Key 1 | NIL     | 2     | P            |
| Key 2 | Value 5 | 4     | U            |
| Key 3 | Value 6 | 1     | U            |



[![../../docs/graph/Fig_011.png](https://github.com/daos-stack/daos/raw/master/docs/graph/Fig_011.png)](https://github.com/daos-stack/daos/blob/master/docs/graph/Fig_011.png)

红黑树和任何传统的二叉树一样，将小于根结点的键组织到左边的子树中，大于根结点的键组织到右边的子树中。值指针和键一起存储在每个节点中。另一方面，基于B+树的索引将键按升序存储在叶子节点，而叶子节点就是存储值的地方。根节点和内部节点(相应地用蓝色和栗色编码)便于定位适当的叶节点。每个B+树节点有多个位置，位置的数量由顺序决定。节点最多可以有1个序号。在红黑树的情况下，容器句柄cookie必须与每个键一起存储，但在B+树的情况下，只有叶节点上的cookie就足够了，因为遍历时不使用cookie。

在下面的[表](https://github.com/daos-stack/daos/tree/master/src/vos#7e)中，n是树中条目的数量，m是键的数量，k是键的数量，两个唯一键之间的epoch条目。

**Comparison of average case computational complexity for index**

| Operation   | Red-black tree          | B+Tree              |
| ----------- | ----------------------- | ------------------- |
| Update      | O(log2n)                | O(logbn)            |
| Lookup      | O(log2n)                | O(logbn)            |
| Delete      | O(log2n)                | O(logbn)            |
| Enumeration | O(m* log2(n) + log2(n)) | O(m * k + logb (n)) |

虽然这两种解决方案都是可行的实现，但确定理想的数据结构将取决于这些数据结构在持久性内存硬件上的性能。

VOS还支持对这些结构的并发访问，这要求所选择的数据结构在并发更新时提供良好的可伸缩性。与B+树相比，红黑树的再平衡对树结构的影响更大;因此，B+树可以提供更好的并发访问性能。此外，由于B+树节点包含许多槽，具体位置取决于每个节点的大小，因此可以更容易地在缓存中预取数据。此外，上述[表](https://github.com/daos-stack/daos/tree/master/src/vos#7e)中的顺序计算复杂性表明，与红黑树相比，基于B+树的合理顺序KV存储可以表现得更好。

VOS支持枚举在给定时间内有效的键。VOS提供了一种基于迭代器的方法来从KV对象中提取所有的键和值。基本上，KV索引是按键排序的，然后按epoch排序的。由于每个键都保存着很长的更新历史，树的大小可能非常大。红黑树枚举的渐进复杂度为O(m* log (n) + log (n))，其中m是在请求的迭代周期内有效的键数。在树中查找第一个元素的时间为O(log2 (n))，查找后继元素的时间为O(log2 (n))。因为需要检索“m”个键，所以这个枚举的复杂度为O(m * log2 (n))。

在B+-树的情况下，叶节点是升序排列的，枚举将直接解析叶节点。其复杂度为O (m * k + logbn)，其中m是一个epoch内有效的键的个数，k是B+树中两个不同键之间的元素个数，B是B+树的顺序。如果两个不同的键之间有"k"个epoch元素，那么时间复杂度为O(m * k)，而定位树中最左边的第一个键则需要额外的O(logbn)。如上面的[图](https://github.com/daos-stack/daos/tree/master/src/vos#7d)所示的通用迭代器接口也将用于KV枚举。

除了枚举一个epoch内有效的对象的键，VOS还支持枚举两个epoch之间修改的对象的键。epoch索引表提供了每个epoch中更新的键。通过聚合与每个epoch相关联的键列表，VOS可以生成一个具有最新epoch的键列表(通过保持键的最新更新并丢弃旧版本)。通过在关联的索引数据结构中查找列表中的每个键，VOS可以使用基于迭代器的方法提取值。



## Key Array Stores

VOS支持的第二种对象是键数组对象。与KV存储类似，数组对象允许多个版本，并且必须能够同时写入、读取和输入字节范围的任何部分。下面的[图](https://github.com/daos-stack/daos/tree/master/src/vos#7f)显示了一个键数组对象中的区段和纪元安排的简单示例。在本例中，不同的线表示存储在各自区段中的实际数据，并对写入该区段范围的不同线程进行颜色编码。

**Example of extents and epochs in a Key Array object**

[![../../docs/graph/Fig_012.png](https://github.com/daos-stack/daos/raw/master/docs/graph/Fig_012.png)](https://github.com/daos-stack/daos/blob/master/docs/graph/Fig_012.png)

在[上面](https://github.com/daos-stack/daos/blob/master/src/vos/7f)的例子中，不同的区段范围之间有显著的重叠。VOS支持最近历元访问(nearest-epoch access)，这要求读取任何给定区间的最新值。例如，在上面的[图](https://github.com/daos-stack/daos/tree/master/src/vos#7f)中，如果对纪元10的区段范围4- 10有一个读请求，那么结果读取缓冲区应该包含纪元9的区段7-10、纪元8的区段5-7和纪元1的区段4-5。VOS数组对象还支持部分范围和完整范围的punch。



**Example Input for Extent Epoch Table**

| Extent Range | Epoch | Write (or) Punch |
| ------------ | ----- | ---------------- |
| 0 - 100      | 1     | Write            |
| 300 - 400    | 2     | Write            |
| 400 - 500    | 3     | Write            |
| 30 - 60      | 10    | Punch            |
| 500 - 600    | 8     | Write            |
| 600 - 700    | 9     | Write            |



r -树提供了一种合理的方式来表示范围和历元有效性范围，从而限制处理读请求所需的搜索空间。VOS提供了一种特殊的r树，称为区间有效性树(Extent-Validity tree, EV-Tree)来存储和查询版本化数组索引。在传统的r树实现中，矩形是有界且不可变的。在VOS中，“矩形”由一个轴上的范围和另一个轴上的纪元有效性范围组成。然而，在插入矩形时，历元的有效范围是未知的，因此所有矩形的插入都是假设上界为无穷大。最初，DAOS设计要求在insert上分裂这种in-tree矩形以限制有效性范围，但由于一些因素导致决定保持原始有效性范围。首先，更新持久性内存的开销比查找高一个数量级。其次，快照之间的重写可以通过聚合删除，从而保持相当小的重叠写入历史。因此，EV-Tree实现了一个由两部分组成的取指算法。

1. 找出所有重叠区段。这将包括请求周期之前发生的所有写操作，即使它们已经被后续的写操作覆盖。
2. 按范围开始，然后按纪元排序
3.遍历有序数组，必要时拆分区段并尽可能地标记它们
4. 重新排序数组。最后的排序可以根据用例选择保留或丢弃空洞和覆盖的区段。

TODO:创建一个新的图形**Rectangles表示extent_range。epoch_validity使用[上面的表](https://github.com/daos-stack/daos/tree/master/src/vos#7g)**

[![../../docs/graph/Fig_016.png](https://github.com/daos-stack/daos/raw/master/docs/graph/Fig_016.png)](https://github.com/daos-stack/daos/blob/master/docs/graph/Fig_016.png)

下图[下图](https://github.com/daos-stack/daos/blob/master/src/vos/7l)显示了使用EV-Tree的分裂和裁剪操作构造的矩形，用于前面[表](https://github.com/daos-stack/daos/tree/master/src/vos#7g)中的示例，在偏移量{0 - 100}处额外写入，以考虑广泛分裂的情况。上图[上图](https://github.com/daos-stack/daos/tree/master/src/vos#7k)显示了相同示例的EV-Tree构造。

**Tree (order - 4) for the example in Table 6 3 (pictorial representation shown in the figure [above](https://github.com/daos-stack/daos/tree/master/src/vos#7g)**

[![../../docs/graph/Fig_017.png](https://github.com/daos-stack/daos/raw/master/docs/graph/Fig_017.png)](https://github.com/daos-stack/daos/blob/master/docs/graph/Fig_017.png)

ev树中的插入通过检查重叠来定位要插入的适当叶节点。如果多个边界框重叠，则选择放大最小的边界框。通过选择面积最小的边界框来解决进一步的连接。每次插入操作的最大开销可能是O (logbn)。

除了上面描述的假重叠问题之外，EV-Tree的搜索工作与R-Tree类似。必须寻找所有重叠的内部节点，直到找到匹配的内部节点和叶节点。由于区间范围可以跨越多个矩形，因此一次搜索可以搜索到多个矩形。在理想情况下(整个区间都落在一个矩形上)，读取开销为O(logbn)，其中b是树的顺序。排序和拆分阶段会增加O(n log n)的额外开销，其中n是匹配的区段数量。在最坏的情况下，这相当于树中的所有区段，但由于聚合和单个键对应的单个分片对应的树相对较小，这种情况会得到缓解。

对于从ev树中删除节点，可以使用与搜索相同的方法来定位节点，并且可以删除节点/槽。一旦叶子节点被删除，为了合并多个表项数小于order/2的叶子节点，需要进行重新插入。EV-tree会进行重新插入(而不是像B+树那样合并叶节点)，因为在删除叶节点/叶槽时，边界框的大小会发生变化，并且确保矩形组织成最小边界框而没有不必要的重叠是很重要的。在VOS中，仅在聚合和丢弃操作时需要delete。这些操作将在下一节中讨论([基于纪元的操作](https://github.com/daos-stack/daos/tree/master/src/vos#74))。



## Conditional Update and MVCC

VOS支持对单个dkey和key进行条件操作。支持以下操作:

- 条件取指:如果键存在则取，否则用- der_nonexist失败
- 条件更新:如果键存在则更新，否则使用- der_nonexist失败
- 条件式插入:如果键不存在则更新，否则使用- der_exist失败
- 条件打孔:如果键存在则打孔，否则以- der_nonexist失败

这些操作提供了原子操作，支持某些需要原子操作的用例。条件操作使用存在检查和读取时间戳的组合来实现。读时间戳使有限的MVCC能够防止读/写竞争，并提供可串行性保证。



### VOS Timestamp Cache

VOS维护一个读写时间戳的内存缓存，以强制MVCC语义。时间戳缓存由两部分组成。

1. 负入口缓存。每个目标的全局数组，包括对象、dkeys和akeys。每层的索引由父实体的索引(如果是容器，则为0)和所述实体的散列值的组合确定。如果两个不同的键映射到同一个索引，则它们共享时间戳条目。这可能会导致一些错误的冲突，但只要有进展，就不会影响正确性。该数组用于存储VOS树中不存在的项的时间戳。创建项后，它将使用下面第2部分描述的机制。请注意，同一个目标中的多个池使用此共享缓存，因此在实体存在之前，也可能出现跨池的虚假冲突。这些条目在启动时使用启动服务器的全局时间进行初始化。这确保了强制重启之前的任何更新，以确保我们保持自动性，因为当服务器宕机时，时间戳数据会丢失。
2. 正入口缓存。每个目标为现有的容器、对象、dkey和akey提供LRU缓存。每个级别使用一个LRU数组，这样容器、对象、dkey和akey只与相同类型的缓存项冲突。当现有项从缓存中移除时，一些准确性会丢失，因为这些值将与上文#1中描述的对应负项合并，直到该项被带回到缓存中。缓存项的索引存储在VOS树中，但它仅在运行时有效。在服务器重启时，LRU缓存从重启发生时的全局时间初始化，所有条目自动失效。在一个新项进入LRU时，使用对应的负项进行初始化。LRU项的索引存储在VOS树中，后续访问的查找时间为O(1)。



### Read Timestamps

时间戳缓存中的每个条目都包含两个读时间戳，以便为DAOS操作提供可串行化保证。这些时间戳是

1. 低时间戳(entity.low)表示根在该实体的子树中* *所有* *节点都已在entity.low处读取
2. 高时间戳(entity.high)表示在以该实体为根的子树中*至少*有一个节点已经在entity.high被读取。

对于任何叶节点(即key)， low == high;对于任何非叶节点，low <= high。

[下面](https://github.com/daos-stack/daos/tree/master/src/vos#824)描述了这些时间戳的用法。



### Write Timestamps

为了检测违反纪元不确定性，VOS还为每个容器、对象、dkey和key维护了一对写时间戳。从逻辑上讲，时间戳表示对实体本身或子树中某个实体的最近两次更新。如果以后有任何更新，则需要至少两个时间戳，以避免假定不确定性。[下图](https://github.com/daos-stack/daos/tree/master/src/vos#8a)显示了至少需要两个时间戳。如果只使用一个时间戳，第一、第二和第三种情况将无法区分，并将作为不确定情况而被拒绝。所有情况下都使用最精确的写时间戳。例如，如果访问是一个数组，我们将检查对应键或对象没有不确定的冲击时的冲突范围。

**Scenarios illustrating utility of write timestamp cache**

[![../../docs/graph/uncertainty.png](https://github.com/daos-stack/daos/raw/master/docs/graph/uncertainty.png)](https://github.com/daos-stack/daos/blob/master/docs/graph/uncertainty.png)



### MVCC Rules

每个DAOS I/O操作都属于一个事务。如果用户没有将操作与事务关联，则DAOS将此操作视为单操作事务。因此，上文定义的条件更新被视为包含条件检查的事务，如果检查通过，则是更新或打卡操作。

每笔交易都有一个纪元。单操作事务和条件更新事务从它们访问的冗余组服务器中获取epoch，快照读事务从快照记录中获取epoch，其他事务从它访问的第一个服务器的HLC中获取epoch。(早期的实现使用客户端HLCs来选择最后一种情况的epoch。为了放松客户端的时钟同步要求，后来的实现转移到使用服务器HLCs来选择epoch，同时引入客户端HLC跟踪器，跟踪客户端听说过的最高服务器HLC时间戳。)事务使用其epoch执行所有操作。

MVCC规则确保事务按照它们的epoch顺序序列化执行，同时确保每个事务在打开之前观察所有冲突事务，只要系统时钟偏移始终在预期的最大系统时钟偏移(epsilon)内。为方便起见，规则将I/O操作分为读和写:

——读
-获取密钥[密钥级别]
-检查对象空值[对象级别]
-检查dkey空[dkey等级]
-检查键空[键级别]
-列出容器下的对象[容器级别]
-在object[对象级别]下列出dkey
-在dkey下列出密钥
-将recx列在key下[key级别]
-在object[对象级别]下查询min/max dkeys
-在dkey下查询min/max akeys [dkey级别]
-在key下查询min/max recx [key级别]
- - -写
-更新密钥[密钥级别]
-敲击键盘[按键级别]
-打孔dkey [dkey等级]
-重击对象[对象级别]

每次读写都是在四个级别中的一个:容器、对象、dkey和key。一个操作被认为是对根在它的层次上的整个子树的访问。尽管这会引入一些错误的冲突(例如，链表操作与不改变链表结果的底层更新操作)，但这种假设简化了规则。

纪元e的读取遵循以下规则:

```
// Epoch uncertainty check
if e is uncertain
    if there is any overlapping, unaborted write in (e, e_orig + epsilon]
        reject

find the highest overlapping, unaborted write in [0, e]
if the write is not committed
    wait for the write to commit or abort
    if aborted
        retry the find skipping this write

// Read timestamp update
for level i from container to the read's level lv
    update i.high
update lv.low
```

A write at epoch e follows these rules:

```
// Epoch uncertainty check
if e is uncertain
    if there is any overlapping, unaborted write in (e, e_orig + epsilon]
        reject

// Read timestamp check
for level i from container to one level above the write
    if (i.low > e) || ((i.low == e) && (other reader @ i.low))
        reject
if (i.high > e) || ((i.high == e) && (other reader @ i.high))
    reject

find if there is any overlapping write at e
if found and from a different transaction
    reject
```

A transaction involving both reads and writes must follow both sets of rules. As optimizations, single-read transactions and snapshot (read) transactions do not need to update read timestamps. Snapshot creations, however, must update the read timestamps as if it is a transaction reading the whole container.

When a transaction is rejected, it restarts with the same transaction ID but a higher epoch. If the epoch becomes higher than the original epoch plus epsilon, the epoch becomes certain, guaranteeing the restarts due to the epoch uncertainty checks are bounded.

Deadlocks among transactions are impossible. A transaction t_1 with epoch e_1 may block a transaction t_2 with epoch e_2 only when t_2 needs to wait for t_1's writes to commit. Since the client caching is used, t_1 must be committing, whereas t_2 may be reading or committing. If t_2 is reading, then e_1 <= e_2. If t_2 is committing, then e_1 < e_2. Suppose there is a cycle of transactions reaching a deadlock. If the cycle includes a committing-committing edge, then the epochs along the cycle must increase and then decrease, causing a contradiction. If all edges are committing-reading, then there must be two such edges together, causing a contradiction that a reading transaction cannot block other transactions. Deadlocks are, therefore, not a concern.

If an entity keeps getting reads with increasing epochs, writes to this entity may keep being rejected due to the entity's ever-increasing read timestamps. Exponential backoffs with randomizations (see d_backoff_seq) have been introduced during daos_tx_restart calls. These are effective for dfs_move workloads, where readers also write.



### Punch propagation

Since conditional operations rely on an emptiness semantic, VOS read operations, particularly listing can be very expensive because they would require potentially reading the subtree to see if the entity is empty or not. In order to alieviate this problem, VOS instead does punch propagation. On a punch operation, the parent tree is read to see if the punch causes it to be empty. If it does, the parent tree is punched as well. Propagation presently stops at the dkey level, meaning the object will not be punched. Punch propagation only applies when punching keys, not values.



## Epoch Based Operations

Epochs provide a way for modifying VOS objects without destroying the history of updates/writes. Each update consumes memory and discarding unused history can help reclaim unused space. VOS provides methods to compact the history of writes/updates and reclaim space in every storage node. VOS also supports rollback of history in case transactions are aborted. The DAOS API timestamp corresponds to a VOS epoch. The API only allows reading either the latest state or from a persistent snapshot, which is simply a reference on a given epoch.

To compact epochs, VOS allows all epochs between snapshots to be aggregated, i.e., the value/extent-data of the latest epoch of any key is always kept over older epochs. This also ensures that merging history does not cause loss of exclusive updates/writes made to an epoch. To rollback history, VOS provides the discard operation.

```
int vos_aggregate(daos_handle_t coh, daos_epoch_range_t *epr);
int vos_discard(daos_handle_t coh, daos_epoch_range_t *epr);
int vos_epoch_flush(daos_handle_t coh, daos_epoch_t epoch);
```

Aggregate and discard operations in VOS accept a range of epochs to be aggregated normally corresponding to ranges between persistent snapshots.



### VOS Discard

Discard forcefully removes epochs without aggregation. This operation is necessary only when the value/extent-data associated with a pair needs to be discarded. During this operation, VOS looks up all objects associated with each cookie in the requested epoch range from the cookie index table and removes the records directly from the respective object trees by looking at their respective epoch validity. DAOS requires a discard to service abort requests. Abort operations require a discard to be synchronous.

During discard, keys and byte-array rectangles need to be searched for nodes/slots whose end-epoch is (discard_epoch - 1). This means that there was an update before the now discarded epoch, and its validity got modified to support near-epoch lookup. This epoch validity of the previous update has to be extended to infinity to ensure future lookups at near-epoch would fetch the last known updated value for the key/extent range.



### VOS Aggregate

During aggregation, VOS must retain the latest update to a key/extent-range discarding the others and any updates visible at a persistent snapshot. VOS can freely remove or consolidate keys or extents so long as it doesn't alter the view visible at the latest timestamp or any persistent snapshot epoch. Aggregation makes use of the vos_iterate API to find both visible and hidden entries between persistent snapshots and removes hidden keys and extents and merges contiguous partial extents to reduce metadata overhead. Aggregation can be an expensive operation but doesn't need to consume cycles on the critical path. A special aggregation ULT processes aggregation, frequently yielding to avoid blocking the continuing I/O.



## VOS Checksum Management

VOS is responsible for storing checksums during an object update and retrieve checksums on an object fetch. Checksums will be stored with other VOS metadata in storage class memory. For Single Value types, a single checksum is stored. For Array Value types, multiple checksums can be stored based on the chunk size.

The **Chunk Size** is defined as the maximum number of bytes of data that a checksum is derived from. While extents are defined in terms of records, the chunk size is defined in terms of bytes. When calculating the number of checksums needed for an extent, the number of records and the record size is needed. Checksums should typically be derived from Chunk Size bytes, however, if the extent is smaller than Chunk Size or an extent is not "Chunk Aligned," then a checksum might be derived from bytes smaller than Chunk Size.

The **Chunk Alignment** will have an absolute offset, not an I/O offset. So even if an extent is exactly, or less than, Chunk Size bytes long, it may have more than one Chunk if it crosses the alignment barrier.

### Configuration

Checksums will be configured for a container when a container is created. Checksum specific properties can be included in the daos_cont_create API. This configuration has not been fully implemented yet, but properties might include checksum type, chunk size, and server side verification.

### Storage

Checksums will be stored in a record(vos_irec_df) or extent(evt_desc) structure for Single Value types and Array Value types respectfully. Because the checksum can be of variable size, depending on the type of checksum configured, the checksum itself will be appended to the end of the structure. The size needed for checksums is included while allocating memory for the persistent structures on SCM (vos_reserve_single/vos_reserve_recx).

The following diagram illustrates the overall VOS layout and where checksums will be stored. Note that the checksum type isn't actually stored in vos_cont_df yet.

[![../../docs/graph/Fig_021.png](https://github.com/daos-stack/daos/raw/master/docs/graph/Fig_021.png)](https://github.com/daos-stack/daos/blob/master/docs/graph/Fig_021.png)

### Checksum VOS Flow (vos_obj_update/vos_obj_fetch)

On update, the checksum(s) are part of the I/O Descriptor. Then, in akey_update_single/akey_update_recx, the checksum buffer pointer is included in the internal structures used for tree updates (vos_rec_bundle for SV and evt_entry_in for EV). As already mentioned, the size of the persistent structure allocated includes the size of the checksum(s). Finally, while storing the record (svt_rec_store) or extent (evt_insert), the checksum(s) are copied to the end of the persistent structure.

On a fetch, the update flow is essentially reversed.

For reference, key junction points in the flows are:

- SV Update: vos_update_end -> akey_update_single -> svt_rec_store
- Sv Fetch: vos_fetch_begin -> akey_fetch_single -> svt_rec_load
- EV Update: vos_update_end -> akey_update_recx -> evt_insert
- EV Fetch: vos_fetch_begin -> akey_fetch_recx -> evt_fill_entry

### Marking data as corrupted

When data is discovered as being corrupted, the bio_addr will be marked with a corrupted flag to prevent subsequent verifications on data that is already known to be corrupted. Because the checksum scrubber will be iterating the vos objects, the vos_iter API is used to mark objects as corrupt. The vos_iter_process() will take the iter handle that the corruptions was discovered on and will call into the btree/evtree to update the durable format structure that contains the bio_addr.



## Metadata Overhead

There is a tool available to estimate the metadata overhead. It is described on the [storage estimator](https://github.com/daos-stack/daos/blob/master/src/client/storage_estimator/README.md) section.



## Replica Consistency

DAOS supports multiple replicas for data high availability. Inconsistency between replicas is possible when a target fails during an update to a replicated object and when concurrent updates are applied on replicated targets in an inconsistent order.

The most intuitive solution to the inconsistency problem is distributed lock (DLM), used by some distributed systems, such as Lustre. For DAOS, a user-space system with powerful, next generation hardware, maintaining distributed locks among multiple, independent application spaces will introduce unacceptable overhead and complexity. DAOS instead uses an optimized two-phase commit transaction to guarantee consistency among replicas.



### Single redundancy group based DAOS Two-Phase Commit (DTX)

When an application wants to modify (update or punch) a multiple replicated object or EC object, the client sends the modification RPC to the leader shard (via [DTX Leader Election](https://github.com/daos-stack/daos/tree/master/src/vos#812) algorithm discussed below). The leader dispatches the RPC to the other related shards, and each shard makes its modification in parallel. Bulk transfers are not forwarded by the leader but rather transferred directly from the client, improving load balance and decreasing latency by utilizing the full client-server bandwidth.

Before modifications are made, a local transaction, called 'DTX', is started on each related shard (both leader and non-leaders) with a client generated DTX identifier that is unique for the modification within the container. All the modifications in a DTX are logged in the DTX transaction table and back references to the table are kept in related modified record. After local modifications are done, each non-leader marks the DTX state as 'prepared' and replies to the leader. The leader sets the DTX state to 'committable' as soon as it has completed its modifications and has received successful replies from all non-leaders. If any shard(s) fail to execute the modification, it will reply to the leader with failure, and the leader will globally abort the DTX. Once the DTX is set by the leader to 'committable' or 'aborted', it replies to the client with the appropriate status.

The client may consider a modification complete as soon as it receives a successful reply from the leader, regardless of whether the DTX is actually 'committed' or not. It is the responsibility of the leader to commit the 'committable' DTX asynchronously. This can happen if the 'committable' count or DTX age exceed some thresholds or the DTX is piggybacked via other dispatched RPCs due to potential conflict with subsequent modifications.

When an application wants to read something from an object with multiple replicas, the client can send the RPC to any replica. On the server side, if the related DTX has been committed or is committable, the record can be returned to. If the DTX state is prepared, and the replica is not the leader, it will reply to the client telling it to send the RPC to the leader instead. If it is the leader and is in the state 'committed' or 'committable', then such entry is visible to the application. Otherwise, if the DTX on the leader is also 'prepared', then for transactional read, ask the client to wait and retry via returning -DER_INPROGRESS; for non-transactional read, related entry is ignored and the latest committed modification is returned to the client.

If the read operation refers to an EC object and the data read from a data shard (non-leader) has a 'prepared' DTX, the data may be 'committable' on the leader due to the aforementioned asynchronous batched commit mechanism. In such case, the non-leader will refresh related DTX status with the leader. If the DTX status after refresh is 'committed', then related data can be returned to the client; otherwise, if the DTX state is still 'prepared', then for transactional read, ask the client to wait and retry via returning -DER_INPROGRESS; for non-transactional read, related entry is ignored and the latest committed modification is returned to the client.

The DTX model is built inside a DAOS container. Each container maintains its own DTX table that is organized as two B+trees in SCM: one for active DTXs and the other for committed DTXs. The following diagram represents the modification of a replicated object under the DTX model.

**Modify multiple replicated object under DTX model**

[![../../docs/graph/Fig_066.png](https://github.com/daos-stack/daos/raw/master/docs/graph/Fig_066.png)](https://github.com/daos-stack/daos/blob/master/docs/graph/Fig_066.png)



### Single redundancy group based DTX Leader Election

In single redundancy group based DTX model, the leader selection is done for each object or dkey following these general guidelines:

R1: When different replicated objects share the same redundancy group, the same leader should not be used for each object.

R2: When a replicated object with multiple DKEYs span multiple redundancy groups, the leaders in different redundancy groups should be on different servers.

R3: Servers that fail frequently should be avoided in leader selection to avoid frequent leader migration.

R4: For EC object, the leader will be one of the parity nodes within current redundancy group.