# OLAP

* 对象存储、块存储、文件存储（表格存储）
* 结构化数据、半结构化数据、非结构化数据
* ap存储引擎，ap查询引擎
* flink、clickhouse、snowflake、[Dremel](https://searchdatabase.techtarget.com.cn/7-20806/)、doris、influxdb（阿帕奇和cncf的新项目）
* Apache Drill、Apache Kylin、Presto、Vertica
* [https://github.com/datafuselabs/databend](https://github.com/datafuselabs/databend)(rust写的)
* 《大数据技术原理与应用》



### 论文

查询

* 数据湖查询引擎：Photon: A Fast Query Engine for Lakehouse Systems
* 学习型查询调度：LSched: A Workload-Aware Learned Query Scheduler for Analytical Database SystemsOnline Sketch-based Query Optimization
* 流数据查询调度：Klink: Progress-Aware Scheduling for Streaming Data Systems
* 查询调度：Adaptive NUMA-aware data placement and task scheduling for analytical workloads in main-memory column-stores
* MonetDB/X100: Hyper-Pipelining Query Execution, Boncz et al., 2005
* Vectorwise: a Vectorized Analytical DBMS
* Multi-query Optimization for On-Line Analytical Processing
* 博客：[https://www.sciencegate.app/keyword/910106](https://www.sciencegate.app/keyword/910106)
* Online Aggregation based Approximate Query Processing: A Literature Survey



索引

* "Bridging the Archipelago between Row-Stores and Column-Stores for Hybrid Workloads" by He et al. (2017)
* Y. Li, et al., [BitWeaving: Fast Scans for Main Memory Data Processing](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/li-sigmod2013.pdf), in _SIGMOD_, 2013 _(Optional)_
* J. Rao, et al., [Cache Conscious Indexing for Decision-Support in Main Memory](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/rao-vldb97.pdf), in _VLDB_, 1999 _(Optional)_
* L. Sidirourgos, et al., [Column Imprints: A Secondary Index Structure](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/p893-sidirourgos.pdf), in _SIGMOD_, 2013 _(Optional)_
* P.-A. Larson, et al., [SQL Server Column Store Indexes](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/p1177-larson.pdf), in _SIGMOD_, 2011 _(Optional)_
* C.Y. Chan, et al., [Bitmap Index Design and Evaluation](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/p355-chan.pdf), in _SIGMOD_, 1998 _(Optional)_
* B. Hentschel, et al., [Column Sketches: A Scan Accelerator for Rapid and Robust Predicate Evaluation](https://15721.courses.cs.cmu.edu/spring2023/papers/04-olapindexes/hentschel-sigmod18.pdf), in _SIGMOD_, 2018
* FastBit: An Efficient Compressed Bitmap Index Technology, Wu et al., 2006
