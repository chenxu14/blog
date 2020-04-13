---
layout: default
title:  "图数据库初步调研(JanusGraph&Nebula)"
---

# 数据模型
------
### nebula
采用邻接表的方式对图数据进行组织，便于快速获取与顶点相邻的边。
  1. 每个顶点通过一个KV来存储，对应的key结构如下(占用24个字节)<br>
     ![nebula_vertex](/images/graphdb/nebula_vertex.png)<br>
     每个字段的描述如下：<br>
     (1) type, 用于标识是数据还是索引(data, index, system, etc)<br>
     (2) partId, 顶点所散列到的分区ID<br>
     >nebula是通过静态hash的方式将顶点散列到不同的分区上，因此分区数一旦确定以后不能在调整，否则会打乱hash结果。<br>
     
     (3) timestamp, 不暴露给user，用于mvcc特性。<br>
     (4) TagID, 顶点对应的Tag。当顶点有多个Tag时需要通过多个KV来存储，每个KV对应一个Tag<br>
     与顶点相关的属性值保存在Value中(TODO 序列化及反序列化逻辑)
  2. 每条边会切割成两个顶点，通过两个KV来存储。<br>
     其中out-edge与源节点的分区相同，in-edge与目标节点的分区相同，这样与目标顶点相关联的边会处在同一个分区里，在执行获取邻接顶点操作时不会出现跨分区查询的情况源顶点边(out-edge)，key的组织结构如下(占用40个字节)<br>
     ![nebula_edge_src](/images/graphdb/nebula_edge_src.png)<br>
     目标顶点边(in-edge)，key的组织结构如下<br>
     ![nebula_edge_src](/images/graphdb/nebula_edge_target.png)<br>
     其中edgeType如果是正整数表示KV对应的是out-edge，如果是负数则对应的是in-edge；rank可用来区分平行边的情况。
     
### JanusGraph
同样采用邻接表的方式对图数据进行组织，除了提供按边切割的能力，JanusGraph还提供按节点切隔的能力，即将顶点和其对应的邻接表打散存储到多个分区上来解决单顶点访问热点问题。如果与HBase的KV存储模型相对应，则每个顶点ID相当于是rowkey，顶点的属性和边都是通过cell来进行存储的，对应的cell结构如下图。

![janusGraph_model](/images/graphdb/janusGraph_model.png)
  1. Edge属性描述<br>
     (1) labelId + direction, label用于标识边的类型(相当于nebula的EdgeType)，最后一位(direction)用来标识边的方向。<br>
     (2) sort key, 定义edge label时为label设置的属性值？便于将同类型的边排序到一块。<br>
     (3) adjacent vertex id, 目标顶点的ID。<br>
     (4) edgeId, 当前边的唯一主键标识。<br>
     (5) signatureKey, defined by the edge label, 作用是？<br>
     (6) otherProps, 保存edge的属性信息。
  2. Property属性描述<br>
     (1) keyId, propertyKey对应的唯一标识(定义PropertySchema的时候JanusGraph自动分配？)<br>
     (2) propertyId, 当property是集合类型时，通过id标识索引。<br>

>从数据模型来看janusGraph针对顶点和其对应的邻接顶点是保存在同一行数据里面的，如果与目标顶点相连的边比较多，并且这些边又归属不同label的话，针对指定label的边进行检索效率可能存在下降，按照EdgeLable将数据拆分成多行似乎更加合理？另外当邻接节点较多时可能需要考虑对顶点进行拆分处理，防止单行数据量过大导致的服务端GC问题。

# 体系结构
------
### nebula
![nebula_arch](/images/graphdb/nebula_arch.png)

基于计算存储相分离的架构设计，计算层通过QueryEngine提供无状态的查询服务，存储层默认采用RAFT + RocksDB的方式做数据管理。
  1. QueryEngine<br>
     主要用来解析客户端发送过来的nGQL，并生成对应的执行计划，同时内置了一些优化器提供如下能力支持。<br>
     (1) 针对一些过滤和聚合操作可以下推到持久层，降低数据传输带来的消耗。<br>
     (2) 流水线优化。<br>
     除此之外QueryEngine引擎还提供ACL访问权限过滤功能，整个QueryEngine的大致工作流程如下图所示。<br>
     ![nebula_queryengine](/images/graphdb/nebula_queryengine.png)<br>
     1. Parser<br>
        用于解析nGQL，生成抽象语法树(语法文件保存在src/parser)
     2. ExecutionPlanner, 根据抽象语法树生成执行计划(a list of actions)，典型的action比如：<br>
        (1) 取一个顶点的所有邻接点。<br>
        (2) 获取某个边的属性。<br>
        (3) 根据指定条件对顶点和边进行过滤。
     3. Optimization Framework<br>
        执行优化器用来对整个执行计划进行优化
     4. Execution<br>
        每个Executor用来执行一个Plan，执行期间可能需要与MetaService和StoreageEngine进行交互(根据Action的操作类型来决定)<br>
  2. StorageService<br>
     存储层的StorageService主要提供了以下功能。
     1. 基于raft做一致性同步<br>
        每个分区对应一个raft分组，分组之间的数据同步共享传输层，并采用统一线程池处理<br>
        Diff Raft Group will share the same transport layer and the same thread pool
     2. 基于shared-nothing的分布式架构，计算/存储能力可由机器数量来做线性扩充。
     3. 将图语义变成了 key-value 语义交给下层的StorageEngine。<br>
        StorageEngine是最小的物理存储单元，每块盘对应一个StorageEngine(封装一个rocksdb实例?)，而partition是最小的逻辑单元，其存储在哪个StorageEngine通过partID%engine_size计算得出。StorageEngine默认基于rocksdb来存储，但rocksdb自己不在写WAL，而是通过raft日志来做数据回放。
     4. 支持批处理操作以提高吞吐。
     5. 不同租户之间的物理资源可做到隔离。<br>
        不同的space将会采用不同的RocksDB实例。
  3. MetaService<br>
     MetaService主要用来提供如下功能：用户管理、schema管理、切片管理、命名空间管理以及Server监听作用类似于HDFS的namenode。

### JanusGraph
![janusGraph_arch](/images/graphdb/janusGraph_arch.png)

由体系架构来看，除了提供OLTP功能之外，JanusGraph还提供OLAP的能力支持。
  1. 存储和索引层<br>
     KV存储方面支持多种存储引擎，包括cassandra和HBase，如果是单机应用还可采用BerkeleyDB来做KV存储，同时通过集成SOLR或者ES还可实现外部索引支持。
  2. 数据库一层主要提供了事务管理，数据管理以及一些查询优化器<br>
     疑问：与nebula相比较相当于是nebula的QueryEngine一层？
  3. 在数据库的上一层主要封装了一些用于交互的API<br>
     基于这些API可完成schema管理，CURD等操作。

除了通过GremlinServer提供查询服务，JanusGraph还可通过嵌入的方式集成到业务自己的JVM中去使用，但是会加重客户端的工作负荷。

>在CAP模型方面，Nebula主要基于RAFT写rocksdb的方式来实现CP，JanusGraph可配合Cassandra实现AP，或者搭配HBase实现CP
在CP表现能力上，nebula的故障恢复时间要快一些，而hbase需要有很长时间的MTTR过程。<br>
另外nebula支持计算能力下推功能，针对一些过滤和聚合操作可以下推到数据层去处理，JanusGraph目前还不具备。
在大查询以及慢查询的处理上，nebula原生提供隔离功能，JanusGraph需要配合底层做一些调整。

# 图遍历语言
------
针对图遍历语言目前还不存在统一的标准(GQL标准正在制定过程中)，普遍采用的语言有gremlin，cypher，nGQL.

### nebula
nebula采用nGQL作为图遍历语言，语法同SQL类似，但是不支持嵌套查询，而是采用管道的方式进行处理。
  1. DML操作语句<br>
     1. 创建NS<br>
        `CREATE SPACE IF NOT EXISTS nba(partition_num=10, replica_factor=1);`<br>
        `USE nba;`<br>
        `SHOW SPACES;`<br>
        `SHOW PARTS`;
     2. 创建TAG<br>
        `CREATE TAG icecream(made timestamp, temperature int) TTL_DURATION = 100, TTL_COL = made;`<br>
        `SHOW TAGS;`<br>
        `DESCRIBE TAG icecream`;
     3. 创建EdgeType<br>
        `CREATE EDGE serve(start_year int, end_year int);`<br>
        `SHOW EDGES;`<br>
        `DESCRIBE EDGE serve`;
     4. Alter操作<br>
        执行alter操作时会自动检查索引，如果要修改的字段是索引列，操作会rejected<br>
        `ALTER EDGE e1 ADD (prop1 int, prop2 string), CHANGE (prop3 string), DROP (prop4, prop5);`<br>
        `ALTER EDGE e1 TTL_DURATION = 2, TTL_COL = prop1;`
     5. drop操作<br>
        drop操作只删除shcema，而数据的删除是在做整理操作时进行的。<br>
        `DROP EDGE  IF EXISTS name;`<br>
        `DROP SPACE IF EXISTS <space_name>;`<br>
        `DROP TAG IF EXISTS <tag_name>;`
  2. DML操作语句<br>
     1. 插入顶点<br>
        `INSERT VERTEX player(name, age) VALUES 100:("Tim Duncan", 42), 101:("Tony Parker", 36), 102:("LaMarcus Aldridge", 33);`<br>
        也可以为顶点指定多个Tag<br>
        `INSERT VERTEX  T1 (i1), T2(s2) VALUES 21: (321, "hello");`
     2. 插入边<br>
        `INSERT EDGE follow(degree) VALUES 102 -> 101@1:(75);` // @指定边对应的rank值<br>
        也可以通过一条语句插入多条边<br>
        `INSERT EDGE follow(degree) VALUES 100 -> 101:(95),100 -> 102:(90),102 -> 101:(75);`
     3. 更新顶点/更新边<br>
        `UPDATE VERTEX 100 SET player.name = "Tim";`<br>
        `UPDATE EDGE 100 -> 101@1 OF follow SET degree = follow.degree + 1`
     4. 删除顶点/删除边<br>
        `DELETE VERTEX 120,121;`<br>
        `DELETE EDGE follow 100 -> 200;`
  3. 查询操作<br>
     1. 通过FETCH PROP ON语句，类似于指定主键的查询方式<br>
        `FETCH PROP ON player 100;` // 指定顶点查询<br>
        `FETCH PROP ON player 100,101,102;` // 指定顶点集合查询<br>
        `FETCH PROP ON serve 100 -> 200;` // 指定边查询
     2. GO FROM OVER语句<br>
        查询目标顶点的出度顶点<br>
        `GO FROM 100 OVER follow YIELD follow._dst AS id | GO FROM $-.id OVER serve YIELD $$.team.name AS Team, $^.player.name AS Player;`<br>
        其中$^表示对源节点的引用，$$表示对目标节点的引用，而$-表示对管道输入的引用，不同于SQL的嵌套查询，nGQL是通过管道来做子查询管理的。
     3. 多跳查询<br>
        `GO 2 STEPS FROM 103 OVER follow;`
     4. 结果去重<br>
        `GO FROM 100,102 OVER serve YIELD DISTINCT $$.team.name AS team_name`
     5. 多边遍历<br>
        `GO FROM 100 OVER follow, serve;`<br>
        `GO FROM 100 OVER *`
     6. 反向查询(体现出有向边的概念)<br>
        `GO FROM 100 OVER follow REVERSELY` // 查找顶点100的入度节点
     7. 双向查询(相当于无向图的处理)<br>
        `GO FROM 102 OVER follow BIDIRECT` // 出度顶点和入度顶点都遍历
  4. 函数支持<br>
     nGQL内置了很多函数的调用支持，包括：比较常用的字符串操作函数，比较函数，聚合函数，LIMIT，ORDER BY以及一些针对集合的操作函数(UNION, INTERSECT, MINUS)，除此之外nGQL也支持自定义函数，便于开发人员引入新的功能和与与计算相关的算法操作(目前原生提供最短路径的算法支持)

### JanusGraph
JanusGraph目前采用gremlin来作为图遍历检索语言，在查询方面需要根据gremlin语句的执行顺序来做一步一步的调用执行，因此不太容易做计算下推处理(不同于nebula可以对整个nGQL生成plan，在根据plan意图做优化处理)。

# 其他相关
------
其他相关特性比对如下：

| facet | JanusGraph | Nebula |
| ---------- | ---------- | ------ |
| __开发语言__ | java | c++ |
| __CAP模型__ | CP / AP | CP |
| __schema__ | yes | 强schema约束 |
| __二级索引__ | 内部索引/外部索引 | 内部索引 |
| __切片方式__ | Uniform均匀分布算法 | 静态hash |
| __OLAP能力支持__ | 是 | 否 |
| __客户端支持__ | java，python | C++，java，Go，python |
| __事务支持__ | ACID | ACID |