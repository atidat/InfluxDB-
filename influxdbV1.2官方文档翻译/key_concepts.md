## 关键概念

在深入研究InfluxDB前，有必要熟悉下一些InfluxDB数据库概念，本文档对这些概念和通用术语做了简单的介绍。

### 样本数据

下一章节引用了下面打印的数据。虽然数据是假的，但可以证明InfluxDB的可靠性。这些数据表示：从2015.8.18午夜到2015.8.18 6:12，地点1的科学家1和地点2的科学家2统计的蝴蝶和蜜蜂的数量。

假设这些数据存储在my_database中，并满足autogen的保留策略
\-------------------------------------  
time&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="tooltip" data-tooltip-text="Field key">butterflies</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="tooltip" data-tooltip-text="Field key">honeybees</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="tooltip" data-tooltip-text="Tag key">location</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="tooltip" data-tooltip-text="Tag key">scientist</span>  
2015-08-18T00:00:00Z&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;12&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;23&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;langstroth  
2015-08-18T00:00:00Z&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;30&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;perpetua  
2015-08-18T00:06:00Z&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;11&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;28&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;langstroth  
<span class="tooltip" data-tooltip-text="Timestamp">2015-08-18T00:06:00Z</span>&nbsp;&nbsp;&nbsp;<span class="tooltip" data-tooltip-text="Field value">3</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="tooltip" data-tooltip-text="Field value">28</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="tooltip" data-tooltip-text="Tag value">1</span>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<span class="tooltip" data-tooltip-text="Tag value">perpetua</span>  
2015-08-18T05:54:00Z&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2&nbsp;	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;11&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;langstroth  
2015-08-18T06:00:00Z&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;10	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;langstroth  
2015-08-18T06:06:00Z&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;8	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;23&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;perpetua  
2015-08-18T06:12:00Z&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;7	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;22	&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;perpetua  



### 讨论

本章节将会解释上面的样本数据的含义

InfluxDB是个时序数据库，所以从时间的角度说，InfluxDB所作的一切都解释的通了。

time column -- InfluxDB中的所有数据都有这一列，存储时间戳信息

butterflies、honeybees column -- fields，fields由field key和field value组成。field key类型为字符串，存储元数据；field value存储具体数据，类型可以为字符串、浮点型数据和布尔值，因为InfluxDB是时序数据库，所以field value通常和时间戳关联。
上面的field key和field value组成了field set集合。fields对于InfluxDB是至关重要了，换句话说，你不可能在没有fields的InfluxDB存储数据。
同样，fields不会被索引。

使用field value作为过滤条件的查询，会扫描所有满足过滤条件的field value。所以，这种查询的性能要比标签查询差。通常，fields不应该存储经常会使用的元数据信息。

最后两列location和scientist就是tags。tags有tag key和tag value组成，它们的类型为字符串。

tag set是tag value的组合，如key1中的val1和key2中的val2是一种tag set。

也许你的数据结构中不需要tags，但是使用它是有绝对好处的，因为它会被索引，所以标签查询的性能更优，可以存储常用的查询数据。

--------

加索引的重要性：表的研究

可以看到下面的两组查询聚焦于honeybees和butterflies：

select * from "census" where "butterflies" = 1

select * from "census" where "honeybees" = 23

因为fields不被索引，所以InfluxDB扫描每个butterflies或每个honeybees的值。显然，这种做法的性能很差，尤其是大表扫描。

为了优化查询，需要重新设计表结构，比如令butterflies和honeybees做为标签

----------

measurement: measurement表现行为像是tags、fields和time列的容器。measurement名称是相关联的fields的数据的描述，类型是字符串，类比于其他sql，measurement的概念类似于表。

每个measurement可以隶属于多个不同的retention policies。

retention policies：描述了InfluxDB存储的数据的时间有效性和集群模式下的副本数量。

census measurement中的所有数据中都自动生成retention policy，既有效期为无穷，副本数为1。

point：相同series，相同时间戳地field set

如例：

```
name: census
time	butterflies		honeybees	location	scientist
2015-08-18T00:00:00z	1	30	1	perpetua
```

这个例子中的series是自动生成的retention policy，measurement是census，tag set是(location = 1, scientist = perputa)。这就是条point，这条point的时间戳是2015-08-18T00:00:00Z。所以，可以对比其他SQL，point就是一条记录。



上面讨论的所有东西都存储于数据库my_database中。InfluxDB中的数据库，和其他传统关系型数据库类似，作为一个逻辑容器，提供users，retention policy、可持续查询和（TODO）时间序列数据能力。

InfluxDB是个schemaless类型数据库，这就意味着可以随时随地地添加measurements、tags、fields，InfluxDB诞生的意义就是如此。



### 回顾与总结

回顾：上面的翻译，自己看了遍后，完全不知道在讲什么。

总结：下述为网上找的资料https://www.cnblogs.com/buttercup/p/15204096.html，关于fields、tags、tag set、serial、measurement、point概念的含义。

时序数据库：专门用于存储时序数据的数据管理系统，其主要设计思想如下：

- 使用特殊设计的外存索引来组织数据
- 强制使用timestamp作为唯一的外键
- 不检查写冲突，避免枷锁，提高写性能
- 对按时间顺序写优化，提高写性能
- 不支持细粒度的数据删除功能，提高读性能
- 牺牲强一致性，提高查询吞吐量，提高读性能
- 提供基于时间窗口的OLAP操作，放弃关联查询功能
- 通过无模式schemaless使系统更容易水平扩展



时序数据库模型（和关系数据库模型的对比）：

| 关系数据库模型 | 时序数据库模型     | 含义                                     |
| -------------- | :----------------- | ---------------------------------------- |
| table          | metric/measurement | 表 -> 指标                               |
| column         | value/field        | 列 -> 值（无索引列）                     |
| index          | tag                | 索引 -> 标签（索引列）                   |
| row            | point              | 行 -> 数据点（时间序列中某个时刻的数据） |
| primary key    | timestamp          | 主键 -> 时间戳（时间序列唯一标识）       |

关于tag：

- tag是字符串类型的键值对
- tag是元数据，而非时序数据
- tag的唯一组合(tag set)确定一个时间序列
- tag可以方便实现粗粒度的聚合
- tag决定索引的组织形式
- tag组合的数量不宜过多

| 概念             | 含义                             |
| ---------------- | -------------------------------- |
| database         | 命名空间，相互隔离               |
| retention policy | 保存策略，定义数据生命周期       |
| bucket           | database + retention policy      |
| measurement      | 相关时间序列集合                 |
| tag              | 标签索引，索引列                 |
| field            | 时序数据，无索引列               |
| point            | 数据点，时间序列中某个时刻的数据 |
| time             | 时间戳，时间序列内唯一标识       |

关于保留策略retention policy：

- 用于管理数据生命周期，包含持续时间duration、副本个数replication factor和分片粒度hard duration
- duration：指定数据保留时间，过期的数据会自动删除
- replication factor：指定集群模式下，需要维护数据副本的个数
- hard duration：指定shard group（TODO）的时间跨度（影响数据分片大小）

保留策略和database的关系：

- 一个database有多个保留策略，每个策略只属于一个database
- 创建point可以指定保留策略，一个measurement有多个保留策略



关于point

在InfluxDB中，point类似于mysql中的一条记录，由四个组件构成：measurement、tag set、field set和timestamp，每个point根据timestamp + series保证唯一性。

关于point可以怎么理解呢？因为InfluxDB是时序数据库，简单来讲就是每个数据都是时间轴上的一个点，这些数据与时间强相关，其中的**tag用来检索**，**field用来记录一些信息**，**measurement用来将相同类型的数据归集**



关于时间序列series：

| series key         | measurement + tag + field key      |
| ------------------ | ---------------------------------- |
| series             | 时间序列，serial key相同的数据集合 |
| series cardinality | 序列基数，series key的数量         |

series key唯一标识一个时间序列，每个数据点都有series key，series key相同的数据点属于同个时间序列，存储在同一个文件中。

series cardinality序列基数是个反馈当前数据的内存压力的性能指标，InfluxDB为每个series key在内存中维护一个索引记录。

注：上述关于series的描述不太好理解，可以参考https://www.cnblogs.com/yihuihui/p/11386679.html



关于查询语言：

- InfluxQL：类SQL的声明式查询语言，同时具有DDL和DML功能
- Flux：函数式查询语言，仅支持DML功能，支持复杂查询，不支持增删改

```sql
InfluxQL示例：
# 创建数据库
CREATE DATABASE "sample_data"
USE sample_data

# 插入样例数据
INSERT census, location=1,scientist=langstroth butterflies=12i,honeybees=23i 1439827200000000000

# 显示数据库中的表
SHOW MEASUREMENTS

# 显示数据库中的field key
SHOW FIELD KEYS

# 显示数据库中的tag key
SHOW TAG KEYS

# 显示数据库中的tag value
SHOW TAG VALUES WITH KEY = scientist

# 查询所有数据
SELECT* FORM census

# 数据求和
SELECT SUM(butterflies) AS butterflies, SUM(honeybees) AS honeybees FROM census WHERE location = '1'

# 数据删除
DELETE FROM census WHERE location = '1'

# 数据更新
INSERT census,location=2,scientist=perpetua butterflies=10i,honeybees=50i 1439849520000000000

# 删除数据库
DROP DATABASE sample_data
```

```shell
curl -X POST 127.0.0.1:8086/api/v2/query -sS \
-H 'Accept:application/csv' \
-H 'Content-type:application/vnd/flux' \
-H 'Authorization: Token root:123456' \
-d '
from(bucket:"sample_data")
|> range(start:time(v: 1439827200000), stop:time(v: 143984952000))
|> filter(fn:(r) => r._measurement == "census" and r.location == "1" and (r._field == "honeybees" or r._field == "butterflies"))
|> limit(n: 100)'
```



关于shard（资料来源https://www.cnblogs.com/ilifeilong/p/12746149.html）

shard是InfluxDB存储引擎的实现（逻辑概念），负责数据的编码存储、读写服务等。将InfluxDB中时间序列化的数据按照时间的先后顺序存入到shard中，每个shard中都负责InfluxDB中一部分的数据存储工作，并以tsm文件的表现形式存储在物理磁盘上，每个存放了数据的shard都属于一个shard group



关于shard、shard group和retention policy（资料来源https://www.cnblogs.com/ilifeilong/p/12746149.html和https://its201.com/article/weixin_36586120/109432886#shard_group__50）

每个retention policy至少有一个shard group，每个shard group至少有一个shard。写入的数据都带有一个时间戳，因此写入时先通过时间判断（range sharding）写入哪个shard group，然后根据series进行hash计算（hash sharding），决定写入哪个shard。

上述实现的好处：

- 一定程度缓解**数据写入热点**问题
- 使得**批量清理过期数据**操作简单
- 提高**数据按时间维度查找**的性能

