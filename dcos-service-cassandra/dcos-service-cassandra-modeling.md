## 数据建模

从关系型数据库RDBMS迁移到Cassandra的一大难点是数据建模。这对已经熟悉了关系模型设计思想和SQL处理风格的开发人员来说尤其痛苦。但是，新的技术应对新的需求，在某些特定场景具有巨大的优势，因此熟悉并掌握这种新的思想对决策者来说仍然是很必要的。

本节主要摘自《Cassandra 3.x High Availablity》一书的第七章，并汇总一些互联网资料。

大多数传统的关系型数据库使用表格方法存储数据，它支持的各种随机访问查询。但是随机磁盘I/O往往是一个显著的瓶颈，因此，为了确保分布式写性能，Cassandra采用**日志结构的存储引擎**，这可以让它将数据顺序写入**提交日志**和Cassandra的持久存储结构**SSTables**。

### 日志结构存储工作机制

Cassandra接收到一次写操作请求时，它会将数据同时写入提交日志和一个称为**memtable**的内存表。提交日志可以确保Cassandra的可靠性，Memtables会周期性的写入磁盘以不可变的SSTables形式保存。

保存在SSTables中的数据拆分为扇区（这些扇区对应着Primary Key）并按列名称排序。这一点非常重要，本节后续部分会详细探讨。提交日志仅在节点重新启动时用于恢复未及时写入SSTables的数据。

这种存储方案的性能有几个与数据建模有重要的影响：

- **写不可变性**

写总是附加操作，更新数据只需要写入新的值并附加一个新的时间戳（每一列都有带有一个时间戳）。

```json
[
{"key": "Jack Jones",
 "cells": [["1:","",1470229749953090],
           ["1:project_name","Cassandra Tuning",1470229749953090],
           ["1:turnover","5000000",1470229749953090],
           ["2:","",1470229928612372],
           ["2:project_name","Spark Layer",1470229928612372],
           ["2:turnover","2000000",1470229928612372]]},
{"key": "Jill Hill",
 "cells": [["1:","",1470229908473768],
           ["1:project_name","Kubernetes Setup",1470229908473768],
           ["1:turnover","2000000",1470229908473768],
           ["2:","",1470229948844042],
           ["2:project_name","Front End",1470229948844042],
           ["2:turnover","1000000",1470229948844042]]}
]
```

- **以最后一次写为准**

如果磁盘上同一列存在多个版本，查询该列时，最新的数据被返回。

- **列无法被物理删除**

不可变性也意味着DELETE操作被执行时，数据并没有被真正被删除。而是该列的值被一个`null`值所覆盖。

- **顺序查询效率最高**

如果查询是顺序读取磁盘上的数据，可以借助底层存储结构的优势从而获得最大化的读性能。通常情况下，Cassandra尽量限制用户使用顺序查询，当然也有例外。

### 理解压缩（Compaction）

Cassandra通过一种称之为压缩（Compaction）的机制来处理随着时间推移而不断膨胀的SSTables。压缩将分散在多个文件中的扇区（partitions）聚合成一个文件，并删除旧的数据，丢弃tombstones。释放空间仅是其中的一个目的，另一个重要的原因是通过将数据转移到一个SSTables中，可以降低跨文件或节点读取Key的磁盘I/O从而显著提高读的性能。

Cassandra提供了多种压缩策略，自3.8（或3.0.8）开始新增了**Time-window**压缩策略用于取代**Date-tiered**压缩策略。

- Size-tiered策略

- Leveled策略

- Time-window策略

### CQL

CQL已经取代Thrift成为与Cassandra交互的标准接口。在了解CQL之前必须要意识到CQL的数据形式并不总是与底层的数据存储结构相匹配，而且，CQL也不是SQL，你必须理解CQL表现形式的真正含义才能避免设计出与Cassandra理念背道而驰的数据模型。

下面详解CQL语句与底层存储的转换。

#### 单个Primary Key

下面是一个名为**books**的表，仅有一个**title**主键：

```
“CREATE TABLE books ( 
   title text, 
   author text, 
   year int, 
   PRIMARY KEY (title) 
);”
```

通过下述语句插入两条数据：

```
INSERT INTO books (title, author, year) VALUES ('Patriot Games', 'Tom Clancy', 1987); 
INSERT INTO books (title, author, year) VALUES ('Without Remorse', 'Tom Clancy', 1993);
```

查询时会得到下述结果：

```
“SELECT * FROM books; 
 
 title           | author     | year 
-----------------+------------+------ 
 Without Remorse | Tom Clancy | 1993 
   Patriot Games | Tom Clancy | 1987”
```

这看上去与传统的ANSI SQL非常相似，但是对应着Cassandra的底层存储却是完全不同。

在存储层，数据用一个Row Key即title和一个name/value组成的列集合表示。每个列都有一个时间戳用于处理冲突。

系统存储时根据Row Key的哈希值在Cassandra集群节点中分布式存储，因此查询返回的结果是无序的。相对比之下，列集合中的数据是根据列名称按自然语言顺序排列的。因此上例中author排在year的前面。**这一点对于构建高效的数据模型至关重要**。

```
Row Key: Without Remorse 
=> (name=author, value=Tom Clancy, timestamp=1393102991499000) 
=> (name=year, value=1993, timestamp=1393102991499000) 
Row Key: Patriot Games 
=> (name=author, value=Tom Clancy, timestamp=1393102991499100) 
=> (name=year, value=1987, timestamp=1393102991499100)
```
**注意**，这是旧的pre-3.0 CLI输出，仅用于理解概念，下述同。 
 

