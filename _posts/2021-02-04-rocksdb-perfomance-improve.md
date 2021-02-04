---
layout: default
title:  "Rocksdb加SPDK改善吞吐能力建设"
---

# 背景说明
------
rocksdb是一个被广泛采用的KV系统，其功能已经逐渐演变成很多上层应用的一个基础组件，像Ceph的bluestore，nebula的点边存储，还有tikv系统，底层都是依赖rocksdb来做数据存储或元数据管理的，因此rocksdb的吞吐能力表现对上层应用系统的能力建设可以起到非常重要的一环。

随着分布式系统开始陆续上云，计算与存储相分离的体系架构也开始广泛得到部署应用，在该体系模式之下存储节点与计算节点往往对物理资源有着不同的需求纬度。存储节点比较侧重磁盘以及带宽，而计算节点则更侧重CPU资源。如果我们能够通过较少的CPU来跑满整个磁盘IO上限，那么便可以预留出更多的CPU资源去服务于计算节点。

基于NVMe设备，一种有效改善吞吐能力的办法是借助内核旁路机制，通过用户态的驱动程序来与底层的NVMe设备进行直接的交互，从而避免系统调用所带来的overhead开销。为此SPDK应运而生，其不但提供了一个用户态的文件系统(blobfs)，还将磁盘驱动程序放到了用户态空间，使得数据的访问可以在用户态直接进行，这样便有效避免了操作系统的上下文切换以及数据在用户态和内核态之间的拷贝传递。

得益于rocksdb对Env的灵活封装，使得底层的数据存储几乎可以是任何的文件系统，包括blobfs。然而通过benchmark我们发现，在某些特定的workload场景下，blobfs的性能表现并没有达到预期的那样理想，比如在对readrandom测试过程中，发现无论是请求时延还是吞吐能力，其表现都是不如内核态的。

![readrandom_test](/images/rocksdb/readrandom_test.png)

为此，我们对blobfs内部的工作流程做了一些调研和梳理，通过分析其内部的执行逻辑来寻找可做改善的性能空间。

# BLOBFS改造 - 多IO Channel支持
------
![blobfs](/images/rocksdb/blobfs.png)

blobfs原生的线程工作模型如上图左侧所示：首先其内部会创建两个不同的io_device(blobfs_md和blobfs_sync)来应对不同类型的IO请求(严格意义上讲是三个，只不过blobfs_io尚未投入使用)，如果请求涉及元数据的访问操作(eg. open, close, create, rename)，需要采用blobfs_md所创建的io_channel，其他情况使用blobfs_sync所创建的io_channel即可。io_channel是blobfs暴露给上层应用的两个通信管道，相当于是交互的入口，然而其本身并不是线程安全的，不同的线程没有办法同时共用同一个io_channel来与blobfs进行交互，只能供单线程内部调用使用。所以SPDK在功能实现上将其与reactor_0进行了绑定，只有reactor_0所对应的线程可以与blobfs做交互，而其他线程或者reactor则需要通过中间队列来做异步中转(即向reactor_0发送事件消息，然后由reactor_0来做统一的处理)。

该线程模型虽然有效规避了meta访问raceCondition的问题，但是在功能实现上也带来了相应的问题短板。
  1. 慢IO拖慢长尾时延更加明显。<br>
     由于所有的IO请求都需要路由给reactor_0，并且IO事件的处理是采用单个qpair来进行的，处理过程中不能出现乱序(io mess up)，这样与多IO链路相比，单链路的慢IO将会影响后续更多的IO请求，从而导致长尾时延得到拖慢。
  2. 跨NUMA通信问题。<br>
     如果reactor_0与reactor_1隶属于不同的NUMA节点，则reactor_1在使用reactor_0所产生的数据时将需要跨NUMA进行通信，从而牺牲一部分性能。
  3. 锁同步产生性能开销。<br>
     锁同步开销主要体现在以下几个方面：<br>
     (1) event队列的出队入队开销，并发线程越多开销会越明显；<br>
     (2) reactor_0处理完IO请求之后需要通过信号量来对调用线程进行通知；<br>
     (3) spdk_fs_request对象池的出队入队开销(IO调用线程从对象池中取元素，reactor_0向对象池中添加元素)。
  4. 横向扩展困难<br>
     在单线程不能跑满IO负荷的情况下，该处理模型没有办法做横向扩展，导致磁盘的带宽资源出现浪费。

为此我们考虑对blobfs原生的IO处理模型进行相应的扩展，来修复以上问题，扩展后的模型如上图右侧所示。首先reactor_0依然保留其原生的处理逻辑，采用blobfs对外暴露的两个io_channel来与其进行交互，所不同的是其他reactor不在将所有的IO请求全部路由给reactor_0进行处理，而是先对IO请求类型进行判断，如果请求涉及元数据的访问操作则将其路由给reactor_0，否则将采用自己独立的io_channel来与blobfs进行直接的交互。体现到rocksdb层面的调用关系便是每当有文件open或close操作时将其路由给reactor_0，而针对文件的读取操作则采用自己独立的io_channel进行。由于rocksdb自身有TableCache机制来缓存每个文件的Reader，因此文件open操作并不是一个频繁的动作。同时rocksdb的文件组织格式sst是没有追加写入的，并且只有当Reader引用计数为0的时候文件才能删除，所以不用担心文件在read过程中发生数据不匹配的情况。

经过以上处理之后，IO的执行链路由一个扩展成了多个，单个慢IO的影响范围将会得到缩减。并且reactor之间不在需要做数据交换(使用其他reactor读取到内存的数据)，有效避免了跨NUMA通信问题。同时锁同步开销也不在是瓶颈，reactor之间不在需要信号量的同步机制，spdk_fs_request对象池的使用也只在线程内部进行，不在需要加锁同步。

# Run-To-Complete模型
------
![run_to_complete](/images/rocksdb/run_to_complete.png)

传统的RPC处理框架在响应客户端请求操作时往往采用基于pipeline的处理方式来进行(如上图左侧所示)，首先通过Listener感知客户端的链接事件，并交由相应的Reader去做通信报文读取，将读取到的信息封装成Call对象保存到请求队列中，以便Handler去做接下来的异步处理。Handler线程在处理过程中如果涉及blobfs的访问操作，还需向reactor线程触发相应的event，以便reactor线程去做对应的IO处理。在整个请求处理链路上需要引入两个中间队列来作为pipeline的串联，引入队列的同时也便产生了相应的overhead，因为元素的出队入队操作需要通过加锁来进行同步。

为了有效规避锁同步所带来的开销，我们可以将整个执行链路改造成基于Run-To-Complete的执行方式(如上图右侧所示)，在该处理方式下，Reader线程和Handler线程不在单独存在，而是将逻辑嵌入到reactor线程里，因此Poller在轮训过程中需要处理以下两方面的事情。
  1. 轮训Reader感知客户端的读写请求，请求不在保存到队列，而是直接调用Handler的处理逻辑；
  2. 轮训qpair感知IO结束事件，并对上层调用进行回溯。

这样便省去了中间队列的中转过程，客户端所发送过来的请求可由当前reactor线程一直处理到最后返回。

# Rocksdb异步化改造
------
![spdk_poller](/images/rocksdb/spdk_poller.png)

基于Run-To-Complete模型改造后，Poller每次轮训的时间片划分可由四个部分组成(如上图左侧所示)
  1. 通过Reader读取客户端发送过来的请求(以NIO的方式进行)
  2. 调用Handler的处理逻辑对读取到的请求进行处理，并将相关的IO请求发送给NVMe设备。
  3. 等待IO运行结束，通过不断的轮训CQ来感知IO运行结束事件。
  4. 将处理结果返回给客户端。

其中比较耗费资源的操作主要集中在步骤3，由于上层应用(rocksdb)不支持异步访问，Poller要一直等待IO运行结束之后才能对上层调用做ACK响应，而在此期间CPU只能空转而不能处理其他事情，这样一方面导致CPU资源出现了极大浪费，另一方面也降低了请求处理的吞吐能力。

而如果上层调用支持异步化访问，则Poller只需要在IO运行结束的时候对其上层调用触发一次callback即可，而不用一直空转来block接下来的请求。处理过程如上图右侧所示：Poller每次轮训只对CQ执行一次poll操作，如果可以从队列中拿到元素，说明有IO运行结束的事件发生，对其上层调用触发callback回溯，否则直接进入下一个轮训周期(即处理下一个请求)，以此来增加请求处理的吞吐能力。为此我们需要对rocksdb的执行链路做一些异步化调整，使其支持异步化访问。

目前针对rocksdb的异步化处理社区主要提供了以下两种方案。<br>
(1) 通过异步线程池来做请求中转；<br>
(2) 基于C++最新的Coroutine特性来做异步改造。<br>
以上两种方式最大的好处是可以不用对rocksdb内核做过多调整，便可以实现异步访问特性，但是缺点和弊端也非常明显：采用方案(1)将打破整个处理链路Run-To-Complete的执行方式，请求处理将再次回归到之前的pipeline模型，从而牺牲一部分性能。而基于Coroutine的处理方式则需要采用最新版本的C++编译器，其与已有框架的兼容性还不得而知，并且与blobfs的poller机制不能形成很好的配合(线程suspend导致poller不能工作)。为此我们考虑对rocksdb的整体请求链路做内核层面的调整，自底向上逐层做callback回溯，部分请求链路的异步化调用关系图如下所示。

### 随机点读异步调用链路
![async_get](/images/rocksdb/async_get.png)

核心流程主要体现在RetrieveBlockAsync的三次调用上(图片中已通过不同颜色进行区分)
  1. 第一次调用主要是用来检索布隆数据，并根据检索到的数据内容来封装FilterBitsReader，然后通过其MayMatch方法判断目标记录是否存在与sst文件中。
  2. 第二次调用主要是用来检索index数据，并根据检索到的数据内容来封装IndexBlockIter，以便通过它来对索引key进行遍历。如果索引key在遍历过程中出现越界(排序大于要检索的key)或者validate校验不合法，会直接触发TableCache#GetAsyncCallback回调，来决定是直接返回还是继续遍历下一个文件。
  3. 最后一次调用主要是根据索引key来定位目标数据块，然后根据数据块的内容来封装DataBlockIter并对其执行SeekForGet操作，将检索到的记录通过GetContext#SaveValue进行保存。

# 其他功能调整
------
除了以上改动之外我们还针对rocksdb的特殊使用场景对blobfs做了其他方面的定制，首先在fsCache层面，blobfs同样是不支持多线程访问的，在原生实现里由于只有reactor_0会去处理IO请求，所以不要求cache的访问具备线程安全特性。然而在增加了多io_channel的能力支持之后，fsCache是有可能被多个reactor线程同时访问的，由此带来raceCondition问题。目前我们临时的解决方案是将cache这一层屏蔽掉(考虑cache的使用可能带来NUMA架构问题)，采用direct_io的方式直接与底层的文件系统进行交互，从而规避线程安全问题。

另外在blobfs的原生实现里，文件的读取操作是需要加自旋锁来进行同步的，防止数据读取过程中发生文件修改的情况，而rocksdb是不会有该场景发生的，因其文件组织格式sst是不支持modify和append操作的，并且在文件的上层还有Reader引用计数，如果计数不为0(即有读取操作)文件是不允许被删除的，因此我们将该spin_lock从整个读取链路中做了移除。

# 性能测试对比
------
### 测试环境
CPU    64 Core   Intel(R) Xeon(R) Gold 5218 CPU @ 2.30GHz <br>
DISK   6.4 T     Non-Volatile memory controller: HGST, Inc. Device 0023

### 测试workload
--statistics=1 --histogram=1 --key_size=16 --value_size=1000 --block_size=4096 --cache_size=0 --bloom_bits=10 --cache_numshardbits=8 --verify_checksum=1 --db=/Nvme1n1/rocksdb --compression_type=none --stats_interval=1000000 --compression_ratio=0 --stats_per_interval=1 --benchmarks=<font color=red>readrandom</font> --duration=600 --use_existing_db=1 --num=<font color=red>2000000000</font>

### 测试情况比较
![readrandom_test2](/images/rocksdb/readrandom_test2.png)

经过以上优化处理之后，rocksdb的吞吐能力得到了非常显著的提升(参见上图左侧)，单线程的吞吐能力提升了<font color=red>11</font>倍(8577提升到90792)，10个线程并发的情况下可以达到<font color=red>60万</font>的qps吞吐。而原生基于内核态的访问方式在16个线程并发的情况下也只能达到129285的吞吐能力，即使所有CPU资源全部利用上也达不到60万QPS的处理上限，同时请求的平均时延也较比内核态有所提升(参见上图右侧)。

# 总结与后续规划
------
以上便是我们有关rocksdb吞吐能力建设所做的一些改进和尝试，核心内容主要围绕在如何与SPDK做有效整合，从而实现kernal旁路机制，以及如何通过异步化访问来提升整体吞吐。目前针对随机点读特性功能已成功落地，范围查询，multiget还有反向scan还在陆续开发过程中，期待后续能与大家做更多的分享。
