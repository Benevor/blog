# F1 Lightning: HTAP as a Service

[https://fuzhe1989.github.io/2020/11/30/f1-lightning-htap-as-a-service/](https://fuzhe1989.github.io/2020/11/30/f1-lightning-htap-as-a-service/)

## 摘要

面临的挑战/问题：one in which there is a great deal of transactional data already existing in several transactional systems, heavily queried by an existing federated engine that does not “own” the transactional systems, supporting both new and legacy applications that demand transparent fast queries and transactions from this combination.

F1 Lightning的目标就是解决这个挑战

## 引言

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

## 现有工作

现有的HTAP系统可以分为两类

* 单一/统一的TP和AP系统
* 分离的TP和AP系统
  * TP/AP共享存储
  * TP/AP解藕存储 (<mark style="color:red;">**lightning**</mark>)

过去的工作表明，使用**行列混存**比使用单一的数据存储模式，在数据的提取和分析上效果更好

lightning不属于单一的TP/AP系统，因为它的分析系统是<mark style="color:red;">**解耦合**</mark>的，这是由于谷歌中原有的TP系统不方便大规模迁移到全新的系统中

共享存储的方案通常需要对OLTP系统进行修改，且通常由TP系统来修改。lightning假设无法更改TP存储。所以采用解藕存储的方案。

许多应用通过松散解藕的离线ETL实现HTAP架构，但是这会导致TP数据和AP副本之间存在高延迟。lightning通过<mark style="color:red;">**CDC**</mark>提取数据，并混合使用存储驻留和磁盘驻留，减小了延迟























