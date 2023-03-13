# F1 Lightning: HTAP as a Service

[https://fuzhe1989.github.io/2020/11/30/f1-lightning-htap-as-a-service/](https://fuzhe1989.github.io/2020/11/30/f1-lightning-htap-as-a-service/)

## 问题

* 统一的TP/AP数据库 与 TP/AP共享存储 的区别在哪里？
* 如果TP数据库中现存的表存在原始数据，怎么通过CDC变更到列存中？

## 摘要

面临的挑战/问题：one in which there is a great deal of transactional data already existing in several transactional systems, heavily queried by an existing federated engine that does not “own” the transactional systems, supporting both new and legacy applications that demand transparent fast queries and transactions from this combination.

F1 Lightning的目标就是解决这个挑战

## 1.引言

现有的工作提出的问题是：what should an ideal HTAP system look like, and what technical advances are needed for good performance.

F1 Lightning提出的<mark style="color:red;">**方案**</mark>是：a loosely coupled HTAP architecture that can support HTAP workloads under a variety of constraints

谷歌现在的系统生态：在新的和旧的工作负载上使用了多种事务数据存储，拥有与这些事务系统松耦合的联合查询引擎

期望的单一HTAP方案：避免数据迁移、允许事务存储系统的设计灵活性、关注点分离（事务系统聚焦于事务处理；查询引擎聚焦于查询处理，尤其是分析型查询）

HTAP-as-a-servic：应用程序只需要将事务存储中的tables标记为“lightning tables”，就可以使用lightning提供的透明的HTAP功能。在生产中使用和支持lightning应该由htap服务商处理，而不是由访问lightning的应用程序或事务系统供应商处理

lightning与联合查询引擎集成：负责创建读优化的副本数据，并保持与事务数据的一致性和新鲜程度、管理受控副本、优化和执行可能跨越事务表和lightning表的查询

lightning的透明性：对于查询引擎的用户和事务存储都是透明的

总述：

* vectorized columnar implementation of merging and compaction
* the use of a two-level schema to provide a great deal of freedom in how Lightning transparently speeds queries over its data
* a change data capture component we call “changepump” that can be extended to new transactional data stores
* methods for reliability and efficiency in a multi-data- center environment
* tight integration and co-design with a federated query engine

## 2.现有工作

### 现有的HTAP系统可以分为两类

* 单一/统一的TP和AP系统
* 分离的TP和AP系统
  * TP/AP共享存储
  * TP/AP解藕存储 (<mark style="color:red;">**lightning**</mark>)

过去的工作表明，使用**行列混存**比使用单一的数据存储模式，在数据的提取和分析上效果更好

lightning不属于单一的TP/AP系统，因为它的分析系统是<mark style="color:red;">**解耦合**</mark>的，这是由于谷歌中原有的TP系统不方便大规模迁移到全新的系统中

共享存储的方案通常需要对OLTP系统进行修改，且通常由TP系统来修改。lightning假设<mark style="color:red;">**无法更改TP存储**</mark>。所以采用解藕存储的方案。

许多应用通过松散解藕的离线<mark style="color:red;">**ETL**</mark>实现HTAP架构，但是这会导致TP数据和AP副本之间存在高延迟。lightning通过<mark style="color:red;">**CDC**</mark>提取数据，并混合使用存储驻留和磁盘驻留，减小了延迟

### 一些与lightning设计相似的系统

* SAP HANA ATR：使用并行日志重放方案来维护以列格式存储的横向扩展OLAP副本
  * tp/ap之间更紧耦合，这使得sap中数据延迟较低。如果一个全新的HTAP系统，选择紧耦合的设计可能会从中受益。
  * 需要修改tp系统，以实现tp引擎内部的日志传送
  * tp运行在单机上，不用考虑分布式日志同步和分布式日志复制
  * SAP HANA ATR使用自己的查询系统用于查询路由，而Lightning使用联邦查询引擎
  * Lightning可以透明地连接多个tp系统，允许用户不迁移数据
* Tiflash：将列存和向量化处理层添加到行存TiDB中，与TiDB查询层紧密集成
  * TiFlash与TiDB之间，仅保持 casual consistency
  * Lightning可以在可查询窗口中提供与源数据库之间的 strong snapshot consistency
* Oracle Database In-Memory
  * “Single System for OLTP and OLAP.”
  * 通过维护活动数据的内存列存储（与持久化行存储保持事务一致）来加速HTAP工作负载
* Middleware-based database replication systems：ETL
  * 复制引擎与底层DBMS系统分离，以执行ETL，从异构数据库中提取数据
  * 这些复制系统没有针对快速分析生成副本进行优化，也没有尝试提供透明的HTAP体验
* Databus：CDC的一种实现
  * 将变更从源数据库中<mark style="color:red;">**增量**</mark>提取到单独的存储中进行分析
  * 与重新导入整个数据集的传统ETL方法相比，CDC通常具有改进的传播延迟和复制效率
  * Databus在精神上类似于Lightning的Changepump。但与Lightning不同，Databus没有为混合事务处理和分析处理提供完整的服务解决方案
  * 相反，它是构建数据副本的组件。我们认为Databus可能是构建完整HTAP解决方案的有用组件
* Wildfire和SnappyData（都是17年的）
  * 旨在使用Spark计算引擎的最新HTAP系统
  * they are still native HTAP systems，需要从现有的tp系统中迁移数据

## 3.系统概述

### 主要组件

* TP数据库：充当数据源，并公开CDC或其他变更复制接口
* F1 Query：分布式SQL查询引擎
  * 联邦查询引擎，支持许多不同的数据源，用户能够编写跨这些系统的无缝连接的查询语句
* Lightning：支持读优化的副本
  * Changepump

<figure><img src="../.gitbook/assets/image (2) (2).png" alt=""><figcaption><p>系统示意图</p></figcaption></figure>

### 建立列存副本的方式：CDC

* 事务处理和分析查询之间存在着很自然的竞争关系。传统的TP系统采用面向行的存储和索引来优化TP负载（写入、点查），导致F1 Query直接在这些存储上进行分析查询效果不佳，虽然可以采用**多workers分布式分析查询**的方式，但是增加了计算资源成本
* 为了应为这些问题，一些团队将TP数据库中的表复制到ColumnIO files进行分析
  * 重复存储（不止两份。多个团队各自保留自己的相同数据副本）
  * ColumnIO文件不支持 in-place update，副本必须作为一个整体定期重述，导致数据新鲜度较差
  * 用户需要显式修改查询语句，以获得副本中的数据，不是透明的
* <mark style="color:red;">**Changepump**</mark>使用F1 DB公开的CDC（或者Spanner的内部日志ship接口）检测更新。将更新转发到单个lightning服务器管理的分区，每个服务器都维护了LSM树，并由分布式文件系统支持，是列存的存储形式。在**不影响事物吞吐量**的情况下，进一步提高了查询性能

### 一致性问题 & 查询重写

* Lightning提供与原始TP数据库之间的<mark style="color:red;">**快照一致性**</mark>
* F1 DB和Spanner都支持使用时间戳的多版本并发控制，提交给Lightning的每个更改都保留其原始提交时间戳
* Lightning保证：特定时间戳的读取将产生与TP数据库在相同时间戳的读取相同的结果
* F1 Query利用这一点，通过<mark style="color:red;">**自动重写**</mark>符合条件的查询来使用Lightning，从而透明地提高查询性能（即当查询特定时间戳数据时，如果数据在Lightning中，则查询改为从Lightning中读取，获得性能改进，并对用户透明；这种查询重写是逐个表进行的，即同一个查询可以既从Lightning中读取表数据，也可以从TP数据库中读取其他表的数据）

### 使用Lightning的效果

* 改进了**分析查询**的资源效率和延迟：Lightning存储的数据针对只读分析查询而不是事务处理进行了优化
* **数据一致性**与**新鲜程度**：分析查询在与OLTP数据库中最新版本一致的快照上运行。更改会自动从OLTP数据库复制，延迟很低。
* **简单的配置**和**重复数据消除**：主要与ETL比较，TP数据库所有者通过简单的配置就可以启用HTAP
* **用户体验透明**：用户无需修改SQL语句，甚至无需知道HTAP副本的存在，即可获得更快的分析查询
* **数据安全**：只有被授权读取原始数据的用户才能使用读取优化副本
* **关注点分离**：Lightning独立于TP系统。使得Lightning可以专注于分析查询的优化；TP数据库可以专注于事务处理
* **可扩展性**：原则上，Lightning可以扩展到在任何提供CDC机制的OLTP数据库上运行

## 4.Lightning架构

### 主要组件

* data storage
  * 在分布式文件系统上存储读优化的文件
  * apply change
  * read data
  * background maintenance operations like data compaction
* change replication
  * 跟踪TP数据库上的事务变更，对其进行分区后分发到对应的数据存储服务器
  * 日志重放
* metadata database
  * 存储表示data storage和change replication组件状态的元数据
* lightning masters
  * 协调其他服务器的操作，并维护Lightning-wide的状态

### 4.1 Read semantics

* Lightning提供<mark style="color:red;">**基于快照隔离的MVCC**</mark>。所有对lightning的query都有一个时间戳，Lightning返回的数据与OLTP数据库中截至该时间戳的数据一致
* 由于Lightning<mark style="color:red;">**异步**</mark>应用TP数据库的变更日志，所以在TP数据库中所做变更对Lightning上的查询可见之前存在一段延迟（Lightning可以设置查询涉及过去多久的上限）
* 安全时间戳：可通过lightning查询的时间戳（最大安全时戳，最小安全时戳）
  * 最大安全时戳：表示Lightning已经接收了该时间戳之前的所有变更
  * 最小安全时戳：表示可以查询的最旧版本的时间戳
  * 查询窗口：Lightning维护的安全时间戳的范围（Google生态中，通常为10小时）
  * Lightning拥有一个从最小安全时间戳开始的数据库单版本快照，以及一个从该时间点到最大安全时间戳的多版本记录（<mark style="color:red;">**？？？**</mark>）

### 4.2 Tables and deltas

* lightning将数据存储在<mark style="color:red;">**Lightning tables**</mark>中。数据库表、索引和视图在lightning中均视为物理表
* 每个Lightning tables都会划分为多个partitions，每个<mark style="color:red;">**partition**</mark>都会存储在<mark style="color:red;">**multi-component LSM-Tree**</mark>中
* <mark style="color:red;">**delta**</mark>：LSM树中的一个component；包含对应lightning表的partial row versions
* <mark style="color:red;">**partial row version**</mark> 由相应行的主键和TP数据库中提交该version的时戳进行标识

Lightning存储的三种<mark style="color:red;">**version**</mark>，对应于对源数据所做的改变：

* Inserts: 包含所有列的值. 每一行的第一个 version 就是一个 insert
* Updates: 更新包含至少一个非主键列的值，并且忽略未修改列的值
* Deletes: 不包含任何非主键列的值，墓碑机制，用于标记特定时戳之后的读不应该读到该行数据

delta维护

* permissive：单个delta可以包含同一个key的多个versions，同一个partition的不同的deltas可以包含相同的version
* 单个delta中，partial row versions由\<key, timestamp>唯一标识，为了加速查找，delta以（key ASC, timestamp DESC）的方式进行排序

### 4.3 Memory-resident deltas

* Lightning提取TP变更后，首先将生成的partial row versions写到一个 memory-resident, row-wise B-tree（牺牲部分读来加速写）
* memory-resident delta最多有两个写入者，有若干读者
* 对于每个partition，都有一个线程负责应用 TP事务日志的变更
* 后台线程定期GC，移除不在查询窗口中的versions
* lightning使用B-Tree中的copy-on-write（？？？）机制，来实现这些并发需求
* 数据一旦被写到Memory-resident deltas，就是对<mark style="color:red;">**query可见**</mark>的了，但要遵守<mark style="color:red;">**Changepump提供的一致性协议**</mark>
* 但是写入内存并不持久，Lightning不维护自己的 write-ahead log（一旦系统断电，内存的delta将会丢失，但是lightning将会通过重放TP日志来恢复）
  * 但是通过事务日志进行几小时变更的重放是不切实际的。因此，lightning定期将内存数据生成checkpoint并持久化到磁盘（原样持久化B树），磁盘中的checkpoint不可读，必须加载到内存
  * 即磁盘中的checkpoint仅是一个备份功能，用于断电恢复
* 当delta太大时，lightning会将其写入磁盘。与checkpoint不同，这个操作包含一个<mark style="color:red;">**转置**</mark>，将内存行存转化为列存
  * 现有的查询将继续从内存驻留的delta读取数据，直到落盘完成，此时，查询将透明地切换到从磁盘（读优化的列存）中读取
  * 这里的磁盘存储是可供查询读取的，且是行存。因为与checkpoint的功能不同

### 4.4 Disk-resident deltas

* 包含了lightning中的大部分数据，且以读优化列式文件格式进行存储
* 每个delta文件存储了两部分数据
  * data part：以PAX的格式进行存储
  * index part：主键上的稀疏B树索引，叶子用来track每个行族的key范围，索引比数据小很多，通常缓存在lightning server中
* 这样的设计是在范围查找和点查中找到了平衡

### 4.5 Delta merging

* Lightning存储的 partial row versions 可能分散在多个delta中（内存或磁盘），因此query需要 merge delta，集合 row version，来获得最终的查询结果（什么时候才要merge？？？）
* Delta merging有两个逻辑步骤
  * merging
    * 删除不同delta中的重复变更，并将不同的version拷贝到新的delta中
    * 由于源delta和新delta不一定使用同一架构，因此merging在copy row时执行。
  * collapsing
    * 将同一个key的多个version合并为同一个version
* 这个过程采用了标准LSM逻辑的向量化版本（？？？），结合了merge和aggregate运算符。
  * 预处理的一步：列出所有的需要参与merge的delta
  * 如果存在主键上的谓词，则可以在合并过程中省略不匹配的version
  * 一旦确定了必须参与合并的delta，Lightning将分两个阶段执行k路合并，这两个阶段将反复应用
    * merge plan generation
      * 从k个输入的每一个中读取一个键块（明确本轮可以collapsed的key范围）
      * 并确定merge plan中需要collapse的version，以及顺序
      * 由于单个主键的多个row versions可以包含在单个delta中，Lightning必须注意它只折叠没有漏洞的完整历史（看例子）
      * collapsing groups、escrow buffer（？？？）
    * merge plan application（？？？）
      * 逐列应用，将行值复制并聚合到适合的缓冲中
      * 然后Lightning刷新输出缓冲区，并将 escrow buffer 用作下一轮的额外输入。也就是说，two-way merge 将具有第三个输入，该输入包含除第一轮之外的所有轮的single row version，但逻辑在其他方面保持不变。

例子：

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption><p>delta merge示例</p></figcaption></figure>

Lightning只能折叠密钥和时间戳小于\<K2, ts : 125>，并且只有其键小于K2的collapsed rows才能被包括在这一轮的输出中。这是因为D1可能具有时间戳在100和125之间的K2的附加版本，但是直到从D1读取下一批才能确定这一点（ts降序排列）。因此，可以在单轮中折叠的version范围的上限是所考虑的所有块中的最大密钥的最小值。



































