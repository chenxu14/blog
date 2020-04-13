---
layout: default
title:  "HBase吞吐能力建设"
---

# 前言
------
公司的hbase集群早先是基于社区1.2.4版本进行搭建的，在时延表现方面起初并不十分理想，受GC尖刺的影响非常严重，针对P99响应时延也只能给业务提供不高于100毫秒的SLA承诺，因此在公司层面接入hbase的业务普遍还是面向近线或者离线场景，而针对时延响应要求比较高的在线业务则没有办法提供能力支持。

近期随着社区补丁的陆续合入，以及公司自研补丁的不断集成，hbase在吞吐能力表现方面已经得到了非常巨大的改善，图计算场景下针对多跳查询已经可以达到3～7倍的能力提升，以下主要是在整个吞吐能力建设过程中，我们所做的一些改进与尝试。

# 合理高效利用缓存
------
HBase原生提供了三种类型的缓存支持，分别是LruBlockCache，BucketCache以及MemcachedBlockCache。其中MemcachedBlockCache主要是借助外部缓存系统来处理相应的块缓存操作，而在hbase内部采用比较多的还是通过组合LruBlockCache和BucketCache来形成一种复合型的缓存模型，即CombinedBlockCache的实现。其中LruBlockCache主要用来缓存索引块和布隆数据块，其数据内容需要保存在堆内；而BucketCache主要用来保存数据块以及LruBlockCache中淘汰的块。不同于LruBlockCache，BucketCache是可以支持多种存储媒介的，比如我们可以将数据保存在堆外，也可以将数据保存到硬盘或者PMEM设备上。即便是将数据保存到硬盘，其对应的访问效率也是要优于HDFS的，因为一方面我们可以利用操作系统的零拷贝功能，另一方面可以避免RPC远程调用以及DN协议带来的开销。所以理想情况下HDFS可以只拿来做容灾备份处理，而数据的访问可以从cache层全部命中，因此需要提供一种大容量的缓存能力支持。

但是缓存容量大了以后有可能会带来以下问题。以公司常用的机器配置模版为例，通常每台机器会挂载12块盘，每块盘提供5T存储，因此每台机器可对外提供约60TB的存储容量。而如果以每个HFileBlock默认采用64KB存储来估算的话，60TB的存储大概需要有近百G的索引块和布隆数据块。由于LruBlockCache是基于堆内进行管理的，如果索引块全部缓存到堆内，将极大增加堆内存的使用开销。另一方面LruBlockCache所管理的缓存数据是需要通过GC来进行回收的，如果空间分配量过小，那么缓存的驱逐频率会更加频繁，随之而来的GC压力也会变得更加明显，尤其在启用cacheOnWrite或者prefetchOnOpen特性时。

既然大数据容量场景下采用LruBlockCache不太能满足我们的需求，那么我们自然会想到能否采用堆外BucketCache来做替换处理，形成一种新的复合型BlockCache，如下图所示:  

![CompositeBucketCache](/images/compositeBucketCache.png)

在此模式下，L1层的BucketCache主要通过堆外内存进行管理，而L2层的BucketCache可通过SSD或PMEM进行管理，以此来解决大容量的缓存需求，同时也意味着我们需要针对BucketCache提供分层存储的能力支持。在功能实现上，分层的BucketCache主要是通过CompositeBucketCache来进行封装的，其延用了原生CombinedBlockCache的处理逻辑，只不过将L1缓存从FirstLevelBlockCache替换成了BucketCache。因此在类结构上我们只需将CombinedBlockCache的代码上移到超类(即CompositeBlockCache)，然后将CompositeBucketCache和CombinedBlockCache分别继承该超类即可(目前代码已提交社区，详细可参考[HBASE-23296](https://issues.apache.org/jira/browse/HBASE-23296))。

### 数据预热处理
有了大容量的缓存能力支撑之后，我们希望把所有的索引块和布隆数据块全部缓存下来，以减少数据在检索过程中对磁盘的seek操作。因为在缓存不命中的情况下，对HFile的读取有可能需要经过3次seek才能定位到目标想要的数据，这将极大降低读取效率。
  1. 第一次seek定位到布隆数据块，用来判断目标记录是否存在于该HFile中。
  2. 第二次seek定位到目标索引块(如果索引有多个层级需要seek多次)。
  3. 第三次根据索引信息定位到目标数据块。

为此我们针对数据写入开启了cacheOnWrite以及prefetchOnOpen特性，并调整了部分缓存的预热逻辑，其中包括：
  1. Region启动加载HFile的过程中，对其大小阈值进行判断，如果大于限定的阈值，缓存其索引块和布隆数据块；而如果没有大于阈值，则将所有的块都缓存下来。<br>
     随着时间的推移和整理操作的不断迭代，历史久远的数据所在的HFile会越来越大，而其访问频率则有可能越来越低，因为大部分业务场景访问的数据都是最新生成的，所以这里我们引入了阈值判断。
  2. 针对整理操作执行同样的处理，确保新HFile生成之前，其数据内容已在缓存中进行了预热。
  3. 针对cacheOnWrite特性优化了内存使用(详细可参考[HBASE-23107](https://issues.apache.org/jira/browse/HBASE-23107))
  4. 针对数据读取操作避免重复预热。<br>
     针对scan类型的查询请求，在检索HFile的过程中一开始是基于pread方式进行读取的(基于统一的Reader流)，当检索数据量达到一定阈值之后需要切换成stream的方式进行读取，在整个切换过程中需要重新构建出Reader实例并对load-on-open区域进行再次预热，这样便带来了无谓的资源使用开销。另外如果启用了prefetchOnOpen特性，相关的数据块还会再次进行预热加载，在缓存使用方面将变得十分不友好，因此针对该问题我们做了相应的补丁修复处理，启用修复后scan的性能得到了将近30%的提升(详细可参考[HBASE-22888](https://issues.apache.org/jira/browse/HBASE-22888))。
  5. 避免cacheOnWrite以及prefetchOnOpen产生数据重复预热(详细可参考[HBASE-23355](https://issues.apache.org/jira/browse/HBASE-23355))。

# 读写链路GC优化
------
针对时延响应要求比较高的java系统，GC往往是最为头疼的问题，如果读写链路有大量的临时对象创建，YGC的执行频率将变得异常频繁。而如果对象的使用空间管理不当，还很容易引发碎片问题，进而增加fullgc的触发频率。所有这些操作都将换来STW，进而影响整个读写链路的吞吐时延。

针对GC问题，一种比较好的改善方式是将占用空间比较大或者使用频率比较高的对象，采用池化的机制来进行管理，然后基于覆写的方式将逻辑上已被释放的空间进行再度利用，从而避免GC层面对象空间的不断申请与释放行为。比如BucketCache针对block的缓存管理方式。

![BucketCache](/images/BucketCache.png)

RS启动过程中会预先分配出block可以使用的内存空间，后续这部分空间将常驻于内存，不参与GC回收。当某个不使用的block被驱逐后，我们可以在逻辑上将其标识为可覆写的状态，这样有后续的block缓存进来时便可以复用这部分空间，而无需在GC层面将其释放回收掉。

在GC能力改善方面，社区在2.0之后的版本已经提供了一些非常优秀的补丁，比如：
  1. HBASE-11425<br>
     将端到端的读取链路offheap化处理，通过池化的机制来管理CellBlock报文的序列化与反序列化操作，并且从BucketCache取块的过程不在需要从堆外拷贝到堆内。
  2. HBASE-15179<br>
     将端到端的写入链路offheap化处理，同时将memstore的chunkpool从堆内移到了堆外，大大缩减了RS进程的堆内存使用开销。
  3. HBASE-14790<br>
     针对WAL的写入提供了扇出的能力支持，同时提供了面向ByteBuffer的写入接口，而不像原生FSHLog只能面向byteArray，这样便有效避免了WAL数据写入需要有堆外拷贝到堆内的过程。
  4. HBASE-14918<br>
     提供了in-memory-flush的能力支持，可周期性的将跳表结构转换成CellChunkMap，来降低ConcurrentSkipListMap带来的overhead开销。
  5. HBASE-21879<br>
     当BlockCache未命中需要从HFile加载目标块时，该补丁为块的加载提供了池化管理功能，避免了每次申请临时空间来构建HFileBlock对象。

以上补丁已经全部backport回我们自己的版本，补丁启用后堆内存空间的使用情况得到了极大的改善，临时对象的申请与释放频率不再那么频繁，YGC的触发频率得到了显著的下降。然而通过对RS进程进行profile发现，整个读写链路的GC优化其实还不够彻底，在很多功能链路上还是遗漏了一些细节，比如：

![gc_optimization](/images/gc_optimization.png)

  1. 客户端向服务端发送put请求时，封装KV数据的CellBlock报文并没有采用池化的机制进行管理，每次需要申请临时的字节数组来封装<br>
     无论是客户端还是服务端都需要有对CellBlock报文执行序列化的操作，服务端主要体现在返回response信息给客户端的过程，而客户端体现在发送request请求到服务端的过程。服务端的序列化处理主要由之前所提到的HBASE-11425来提供，而针对客户端组件还没有提供类似的池化管理功能，为此我们引入了netty的内存池来对其进行管理。<br>
     社区在2.0版本提供了异步RPC功能，并基于netty对客户端代码做了相应重构，因此异步客户端已基于netty内存池对CellBlock做了序列化管理，但是同步客户端尚无此功能，为此我们提交了相关的补丁修复到社区，详细可参考[HBASE-22905](https://issues.apache.org/jira/browse/HBASE-22905)。
  2. 当BucketCache采用SSD来作为存储媒介时(IOEngine为file)，读块操作依然需要有从堆外拷贝到堆内的过程。<br>
     启用基于文件的BucketCache缓存之后，缓存块的读取主要是通过调用FileIOEngine#read来进行的，在对RS进程做profile时发现，有近80%的内存申请是由FileIOEngine#read操作触发的，如图所示: <br>
     ![fileioengine_flame](/images/fileioengine_flame.png)<br>
     为此，针对这部分内存申请，我们延用了HBASE-21879的处理方式，采用池化机制来对其进行管理，功能启用后内存申请操作由80%下降到了5%，gc时延方面得到了近1倍的改良(改善后的火焰图可参考[HBASE-22802](https://issues.apache.org/jira/browse/HBASE-22802))。
  3. 开启CacheOnWrite特性时，块数据的缓存操作需要申请临时字节数组来做数据暂存。<br>
     同HBASE-22802的分析处理过程相类似，我们依然采用池化管理机制来规避这类问题，启用堆外内存池管理之后，临时空间的申请占比由45%下降到了6%，相关的火焰图可参考[HBASE-23107](https://issues.apache.org/jira/browse/HBASE-23107)中的附件。
  4. memstore执行flush操作生成HFile文件时，针对DFSPacket的写入默认同样没有采用池化管理机制，每次都需要申请临时的字节空间。<br>
     针对DFSPacket的池化管理，HDFS已经内置了一个轻量级的内存池管理工具ByteArrayManager#Impl，但是默认是不开启的，为此我们需要在RS端调整dfs.client.write.byte-array-manager.enabled的参数值为true。启用ByteArrayManage之后DFSPacke的内存申请占比从30%下降到了1%。

以上便是有关GC链路的一些优化处理，核心思想主要是采用池化管理机制来降低临时对象的空间申请与释放行为，代码层面主要是通过ByteBuffer池来进行空间管理并配合Unsafe的使用来跳过一些边界检查行为。

# 批量查询加大并发处理粒度
------
在实际应用中，为了提升与服务端的交互能力，我们通常会将多个请求先汇总成一个批次，然后在统一发送到服务端去进行处理，通过降低与服务端的RPC交互频率来换取对应的吞吐能力。典型的应用场景比如图数据库Janusgraph在查询目标顶点的邻接表信息时，便是向服务端发送一个multiget请求。

然而针对该类型的请求(multiget)，服务端并没有提供与之相对应的并发处理模型，请求到达服务端之后针对每个multiget将会采用单一的handler线程来串行处理其中的每一个get，如图所示。

![multiget](/images/multiget.png)

因此，针对批处理请求数量较小但是请求批次很大的场景，服务端资源并不能得到有效充分的利用。为此我们可以针对multiget请求引入一个新的线程池模型，将批次中的每一个get请求分发到对应的线程池中去做处理，以此来增加multiget请求在服务端的并发处理粒度。启用该功能以后，multiget的请求时延可以达到将近40%的性能提升，目前补丁已经提交至社区，相关的代码逻辑可参考[HBASE-23063](https://issues.apache.org/jira/browse/HBASE-23063)。

# YCSB压测情况
------
为了衡量HBase的吞吐能力效果，我们采用了统一基准测试YCSB对集群进行了压测，测试环境如下。
  1. 机器规模
     测试过程中使用了2台RS，每台RS内存分配如下：堆内32G，堆外64G(其中memstore分配15G，L1层的BucketCache分配40G，堆外内存池分配5G)，同时为每台RS挂载一块SSD充当L2缓存，缓存容量为2TB。
  2. 数据规模
     通过YCSB向集群导入20亿行数据，每行10个KV，单KV大小为100个字节，HFileBlock大小设置为32KB，持久化到HDFS之后，单副本存储容量约3TB，由于启用了缓存预热功能，数据导入成功后每台RS的缓存使用如下：L1使用4.3G，L2使用1.6T。

测试过程主要针对multiget请求以及随机get点读两种场景来进行，其中针对multiget请求我们对YCSB做了相应的定制处理，对应的测试结果如下。

### 随机get点读测试
单客户端开启40个线程并发执行1亿次get，测试结果如下。

| ops/sec  | 总用时 (ms) | 平均时延(us) | P99时延(us) | P999时延(ms) | 客户端GC次数 | 客户端GC用时(ms) |
| -------- | ---------- | ----------- | ----------- | ----------- | ----------- | --------------- |
| 46024.58 | 2172752.0  | 863.89      | 1809.0      | 2           | 516         | 3822.0          |

### multiget批量读测试
单客户端开启30个线程并发执行1000万次multiget请求，每个multiget返回50行数据，测试结果如下：

| ops/sec  | 总用时 (ms) | 平均时延(us) | P99时延(us) | P999时延(ms) | 客户端GC次数 | 客户端GC用时(ms) |
| -------- | ---------- | ----------- | ----------- | ----------- | ----------- | --------------- |
| 2657.87  | 3762409.0  | 11227.87    | 16767.0     | 45          | 925         | 7190.0          |

(1)客户端视角的端到端监控如下

![client_monitor](/images/client_monitor.png)

从端到端的监控结果来看，P999时延可以稳定控制在50ms之内，由于每个multiget请求会返回50行数据，因此单行数据(每行10个KV，数据总量1KB)平均下来可达1ms。
(2) 服务端视角的监控如下

![sever_monitor](/images/sever_monitor.png)

从服务端视角来看，单机get吞吐量达到6万时，每秒GC时间平均可控制在6.5毫秒上下，且GC的整体表现非常平稳，P999时延不在受到GC尖刺的影响。