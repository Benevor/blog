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

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption><p>系统示意图</p></figcaption></figure>

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



















