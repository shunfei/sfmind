IndexR: Real-time, Hadoop based Data Warehouse
====
## Summary

IndexR implements a structured data format that can be deployed in a distributed environment, with parallel processing, indexed, and tabular. Based on this data format, IndexR constructs a Data warehouse system ([Data Warehouse](http://baike.baidu.com/view/19711.htm)), which is based on the Hadoop ecosystem, You can do a quick statistical analysis of a massive dataset ([OLAP](http://baike.baidu.com/view/22068.htm)), data can be imported in real time and query without dealy. IndexR is designed to solve problems such as slow analysis, data delay and complex system in large data scenarios. This paper describes IndexR's design ideas, system architecture, and core technical details.

Currently the IndexR project is open source, project address: [https://github.com/shunfei/IndexR](https://github.com/shunfei/IndexR).

## Introduction

Shun Fei Technology, one of the top in program advertising business, docking the whole network of major media, every second produceses millions of analysis data. These data is used in the advertising campaign in the process of a detailed tracking and description, such as the creative display, clicks, the number of activities generated registration, return visits and so on. We need to perform real-time analysis of these data to include customer reports, placement optimization, fraud analysis, billing, etc. The data user's query mode is not fixed, unpredictable, and as the volume of business surges, the volume of data also increases dramatically. We need a new technology to address these needs:

* **Large data sets, low query delay**. The query mode is unpredictable and cannot be estimated; table data is generally more than 100 million, even on the specific queries, filtering conditions may hit a large amount of data, the data in the query will also have a large number of updates, tens of thousands of per second in the data storage. To ensure a lower query delay, in general,considering the query delay requirements within 5s, and for commonly used high-frequency queries within 1s.
* **Real-time**. Data from generation to reflection in the analysis results should only have delay within a few seconds. Timeliness is critical to some businesses, and the more real-time data is, the greater the value.
* **Reliability, consistency, high availability**. This data is one of the company's most important data, any errors and inconsistencies may be directly reflected in the customer report,and the company's business and brand image impact, is critical.
* **Scalable, low-cost, easy to maintain**. The business will develop rapidly, creating new data sources, adding new tables, and old data cannot be deleted, which brings enormous cost pressures, and operational dimension pressure. Typical updates such as index, column-value updates should not affect online services and cannot bring in storage or query latency.
* **SQL Support**. Full support for SQL, as good as MySQL, powerful function. Supports not only common multidimensional analysis, but also complex analytic queries, such as joins, subqueries, and so on, and support custom functions (UDF, UDFA).
* **with the ecological integration of Hadoop**. The vigorous development of Hadoop ecology brings more and more powerful processing ability to large data processing, which can greatly expand the value of the system if combined with the depth of its tool chain.

IndexR data platform group is created to answer these challenges. We are unable to find a tool in the current open source product that meets all of these needs.

Products that offer similar functionality are now available through the use of traditional relational data technologies, or by building cubes to expedite queries. These approaches may pose problems such as operational difficulties, data bottlenecks, or patterns that are not flexible enough to support business change. Some scenarios use memory storage technology, which is expensive to use and does not have a particularly high speed advantage in large data analysis scenarios. In recent years, some time-serie databases have been used to solve some problems in storage delay, but there are still some problems in query performance, usability and scalability.

IndexR Data Warehouse system is based on many excellent open source products, and reference some existing tools which are carefully designed and implemented. It stores data in HDFS, uses zookeeper to communicate and negotiate in the cluster, uses the Hive's convenient management to partition data, through Kafka does high speed real-time  data import, the query layer uses the excellent distributed query engine Apache Drill. Its storage and indexing design refer to Infobright Community Edition and Google Mesa thesis, the compression algorithm borrowed from infobright, and real-time storage from hbase is counsulted to get inspiration.

This paper expounds the IndexR from the following aspects:

* Storage format and indexing, IndexR core modules.
* Real-time storage module to achieve fast storage and query with 0 delay.
* Hierarchy and deployment architecture, how to combine with Hadoop ecosystem deeply.
* Engineering implementation problems and solutions.
* Typical project selection.
* The challenge of the Data warehouse in the new environment, & IndexR's significance.

Currently IndexR at Shun Fei has been in stable operation, supporting the DSP, Web site detection analysis, such as the core business of real-time analysis tasks on daily basis with the cluster of 30 billion + current total data volume.


## Storage format and index design

# data File

IndexR stores structured data, such as the following is a fictitious advertisement put to the user: table A:

Column name | Data type|
-------------|----------|
' Date ' | int |
' Contry ' | string |
' Campaign_id ' | Long |
' Impressions ' | Long |
' Clicks ' | Long |

The data file is called segment, and a segment saves a portion of a table, containing all the columns, as shown below.

! [Segment_file] (segment_file.png)

The segment file is self explanatory and contains version information, full table definitions, metadata for each part (offset), and indexing. In IndexR all columns are indexed by default. The row order can be the natural order of the storage, or it can be sorted by user-defined fields. Such a design simplifies the system architecture, does not require additional metadata storage, is very suitable for parallel processing in a distributed environment, and also facilitates external systems such as Hive's direct use.

Segment row data is further subdivided internally into packs, and each pack has a separate index. The row data inside the pack is stored in columns, that is, the data for a column is centrally stored together. This approach has great advantages for fast traversal of column data, and compression. For the modern universal computer architecture, Cache-friendly, convenient vector process, to give full play to the modern multi-core CPU performance. The Segment column data uses the specially optimized compression algorithm, chooses the different algorithm and the parameter according to the data type, usually the compression rate is above 10:1.

In the actual business data test, IndexR each node can process 100 million of fields per second. Test machine configuration: [Intel (R) Xeon (r) CPU e5-2620 v2 @ 2.10GHz] x 2, 60G RAM, SATA $number RPM DISK. This configuration is lowest in the current server configuration, and a more powerful CPU will have a very large performance boost for IndexR.

# index


IndexR uses rough set indexing ([Rough set index](https://en.wikipedia.org/wiki/Rough_set)), which is able to locate files and locations at very low cost and with high accuracy.

For example, one of our data blocks (packs) has the following data, with date (type int) and Use_name (string) types.

Row ID | Date | User_name |
-------|----------|-----------|
0 | 20170101 | Alice |
1 | 20170101 | Bob |
2 | 20170102 | Henry |
3 | 20170107 | Petter |
4 | 20170110 | Mary |

IndexR has a different indexing method for number and string types, here the basic ideas are described.

For the number type, the maximum (max) value (min) of the column is recorded, and the interval (max-min) is segmented into multiple intervals, each with a bit representation. Then map each specific value into this interval.

bit | Index Chunk | Value |
----|-------------------|-------|
0 | 20170101~20170102 | 1 |
1 | 20170103~20170104 | 0 |
2 | 20170105~20170106 | 0 |
3 | 20170107~20170108 | 1 |
4 | 20170109~20170110 | 1 |

As shown in figure, the value of 1 indicates that there is one or more rows of data in this interval, and 0 means that it does not exist. We only need to store Max, Min, and the value sequence (5 bit) to complete the index of this column.

such as query

>select ' user_name ' from A WHERE ' date ' = ' 20170106 '

Because ' 20170106 ' belongs to interval 2 and value is 0, you can know that ' 20170106 ' does not exist in this pack and can be skipped directly. This is a bloomfilter-like filter, the pack does not hit when does not contain the required data

The index of number and string types are similar, but more complex.

Currently common indexes have a B + Tree index, inverted index, which can be accurately positioned to specific rows, in relatively small amount of data is very effective. Typically, this approach does not have a particularly effective compression, and the data file size is typically between the numbers times of the original data, and when the amount of data expands to a certain extent, the cost of such an index can be magnified and even impossible to serve.

IndexR's rough set index has the advantage of being very fast, the index file is small enough to load into memory in a low cost way, and still works efficiently in the extreme data volume scenario. Because data is usually sorted and clustered, the value base (cardinality) of the column is usually small by observing the actual data, which can effectively filter out extraneous packs. It indexes all of the columns and is ideal for exploratory analysis of business or data analysis scenarios.

## Real-time storage

IndexR supports real-time data additions, but does not support online updates, and can be used offline to update data using tools such as hive, similar to Mesa. It's storage speed is very fast, usually single node single table can reach 30k message/s. After the message arrives at IndexR node, it can be queried immediately.

IndexR's real-time storage module uses a similar lsm-tree structure. Using the Commitlog file to save the message, the latest data is stored in memory and is written to the hard disk after a certain threshold is reached.

! [Realtime_segment] (realtime_segment.png)

In-memory data is periodically stored on the hard disk, and with time will produce more fragmented files, which will be sorted and merged after reaching a certain threshold.

! [Realtime_process] (realtime_process.png)

Rows can be stored in a natural storage order, or they can be sorted by a specified field, similar to the one-level index in a relational database and column Family in HBase, which makes the data more cohesive and advantageous for queries.

Similar to Mesa, if required, IndexR real-time storage can be divided into dimensions (Dimension) and indicators (Metric/measure) based on the concept of Multidimensional Analysis (multidimensional), Rows with the same dimensions are merged together, and the metric uses aggregate functions (aggregation function, E.G. SUM, COUNT), and the table can be designed to be a parent-child relationship of original table.

! [Table_olap] (Table_olap.png)

As shown in figure, table B and table C can be considered a child of table A. Table A has three dimensions (date, country, campaign_id) that can express the most detailed information. Table B and table C reduce the amount of data by reducing the dimension, and can get the results of the query more quickly.

The application layer only needs to do simple table routing, such as

> SELECT ' Date ', ' Country ', SUM (' impressions ') from B WHERE ' country ' = ' CN ' GROUP by ' date ', ' Country '

You can route to table B and get results quickly. If a drill (Drill down) query is required, as

> SELECT ' campaign_id ', SUM (' impressions ') from A WHERE ' country ' = ' CN ' and ' date ' = ' 20170101 ' GROUP by ' campaign_id '

is routed to table A.

This design is similar to the pre-aggregation view in a relational database. In the area of OLAP, especially multidimensional analysis scenarios, this design is very effective.

## Architecture Design

IndexR architecture design follows simple, reliable and extensible principles. It enables large-scale cluster deployments and supports thousands of nodes. In fact, IndexR hardware costs are relatively low and can be extended linearly by adding nodes.

! [Ecosystem] (ecosystem.png)

Apache Drill is IndexR's query layer. Drill is a new query engine focused on SQL computing, using techniques such as code generation, vector processing, column computing, and heap memory (eliminating GC), specifically for the optimization of large datasets. Very fast, and support standard SQL, no migration burden. From our experience of use, it is very stable, engineering quality is very high.

IndexR is mainly responsible for the storage layer, and the specific query process optimization, such as common conditions under the push (predicate pushdown), limit push, and so on, in future will also support the aggregation of the next push (aggregation pushdown). IndexR assigns the computing task to the most suitable node through task assignment algorithm, combining data distance, node busy degree, etc.

HDFS stores specific data files, and Distributed file systems help build node stateless services. Data stored in HDFs can be easily used for other complex analyses using a variety of hadoop tools. We integrated Hive to facilitate off-line processing of the data. Because the data is on the HDFS only, it can be processed simultaneously by a number of tools, eliminating the cumbersome synchronization steps. also 10:1 of the high compression ratio save a huge space.

Data is imported into IndexR by Kafka and other queues. IndexR Real-time import is very flexible and can add or remove import nodes at any time. It has very high import performance (30k/s), thus the pressure of storage delay becomes history.

There is only one node in the IndexR cluster (IndexR nodes), which facilitates deployment and maintenance and does not require partitioning of nodes. Currently IndexR is embedded as a drill plugin in the drillbit process.

! [Deploy_architecture] (deploy_architecture.png)

IndexR provides a IndexR-tool tool that provides a complete operational management tool. For example, can update the table structure online, online add, modify the real-time storage configuration.


## Project Implementation challenges

The algorithm and data structure should be really grounded, and it must be realized through concrete projects, and the quality of project execution determines the final effect of the project. Tall buildings are not built if they have a superb design drawing and no high-quality construction and suitable materials. IndexR works on the most extreme performance, but without losing the flexibility of scalability.

* **Use Direct memory**. IndexR is primarily written using JAVA8, while Java's heap memory (HEAP) and garbage collection (GC) patterns face larger challenges in large data operations scenarios. When large memory is required (over 32G) and data is updated frequently, the problem of the JVM's GC is more obvious, which is prone to performance instability, and the memory model of the object instance is often wasteful. In the IndexR project, we store all the stored data and operation temporary data outside the heap, and manually manage the memory request release. This improves code complexity, but saves more than 1/2 memory from the traditional heap memory pattern, and without the GC cost, an assignment operation involving large amounts of data can typically use memory copies to save a lot of CPU cycles.
* **Make full use of modern CPU capabilities**. IndexR's heap memory model is very useful for fully exploiting the hardware potential, they are usually contiguous memory blocks, no class pointer jumps, no virtual function loss, CPU registers and multilevel caches can be fully utilized, and the use of vector processor is very convenient, there is no structural conversion overhead.
* **Avoid random reads**. Usually the disk is characterized by continuous reads very fast, so Kafka can use disk to do Message Queuing, while random reading is relatively slow, so the bottleneck of traditional database is generally IO. The IndexR index is continuous and read-friendly to the disk, and it reorganizes the data to make it more cohesive. In particular, we have carefully optimized the way file are read.
* **Optimize thread, IO dispatch**. When the task is very busy, CPU scrambles and bring the cost of thread switching which can not be ignored. And because of the particularity of the database environment, in doing busy CPU task, but also network, IO operation, how to do task scheduling, reasonable arrangement of the number of threads and tasks, the overall performance impact is relatively large. Sometimes single-threaded is more efficient than multithreading, and is more resource-saving.
* **Key points can be implemented using C++**. It has obvious advantages in operating efficiency when it involves both memory operation and complex CPU operation scenarios. We put the key performance points, such as compression algorithm, using C++ implementation.

## Tool Selection

IndexR is a new tool that you can consider. use IndexR if your project has the following requirements, or if there have been some selection but not enough to meet the requirements.

Classic scene:

* Need to do a quick statistical analysis on top of the massive data query.
* Requires very fast storage speed and needs real-time analysis.
* Store a large number of historical detail databases. For example, website browsing information, transaction information, security data, power industry data, material networking equipment collection data. This kind of data is usually very large, the data content is complex, storage time is long, and expectation can be more quickly according to various conditions to do the detailed query, or in a certain range to do complex analysis. In this case, we can give full play to the IndexR, which is Low-cost, scalable, suitable for the advantages of large datasets.

Typical selection:

* Use MySQL, PostgreSQL and other relational database, not only for business query (OLTP), also do statistical analysis, generally in the existing business database directly do some analysis requirements. This approach has a performance problem after the volume of data is increased, especially when slow queries can have a significant impact on business queries. You can consider importing data into IndexR for analysis, separating the business database from the analysis database.
* ES, SOLR and other Full-text search databases for statistical analysis scenarios. The biggest feature of this type of database is the use of inverted indexes to solve indexing problems. For statistical analysis scenarios, these are usually not particularly optimized memory and disk pressure in large data scenarios. If you are experiencing performance problems, or if the amount of data is not going to hold, consider using IndexR.
* Coupled, Pinot and so-called time series database. In the case of query conditions hit a large number of data may have performance problems, and the ability to sort, aggregate and so generally not very good, from our experience the operational dimension is difficult, flexibility and scalability is not enough, such as lack of join, subqueries and so on. The hardware resources required to save a large amount of historical data are relatively expensive. This scenario allows you to consider using IndexR direct substitution without worrying about the business implementation problem.
* Infobright, Clickhose and other column database. The column database itself is ideal for OLAP scenarios, and IndexR is also part of a column database. The biggest difference is that IndexR is based on the Hadoop ecosystem.
* Off-line pre-polymerization, building cube, the results of data stored in HBase and other KV databases, such as Kylin. This is very effective in situations where only multidimensional analysis scenarios and queries are relatively simple. The problem is lack of flexibility (flexibility), inability to exploratory analysis, and more complex analysis requirements. IndexR can be configured to achieve the effect of a pre-aggregation, and aggregation is real-time, no delay, can retain the original data or high dimensional data, through the table routing to determine the specific query table.
* To solve the problem of real-time analysis of large amount of data, the upper layer uses Impala, Presto, Sparksql, drill and other computing engines to do queries, and the storage layer uses open source data formats such as parquet, based on the Hadoop ecology. Such architectures are similar to IndexR. IndexR's advantage lies in more efficient index design, better performance, and support for real-time storage, second-level latency. In the same environment as parquet format query performance comparison wise IndexR query speed increase number of times. After IndexR gone through a lot of performance optimization, it is expected to have a better performance.
* Kudu, Phoenix and so on both support OLTP scenarios, but also for the OLAP scene optimization and other open source products. It is often difficult to take the two into account, it is suggested that the real-time library and history library, for different data characteristics of the use of unused storage solutions.
* Memory database. Can only be used as ephimeral storage.

The Cleanse Data platform group has a wealth of experience with most of the technical choices mentioned above, that is, the tools we have used in the build environment, or have had in-depth research and testing, this has prompted the birth of IndexR.

## thinking and summarizing

Large data after recent years of rapid development, complete ecological gradually mature, has long been from the era of Hadoop running MR Jobs. After satisfying the need to analyze a large number of data sets, people gradually put forward higher requirements for timeliness and ease of use, so new tools such as Storm and Spark were born. New challenges are emerging and new opportunities are offered. and traditional Data Warehouse products, in the face of large data impact appears very powerless. IndexR provides a new way to solve this situation.

IndexR is a new generation of data Warehouse system, designed for OLAP scenarios, can host large number of structured data for rapid analysis, support for fast real-time storage. It is powerful, simple and reliable, and supports large-scale cluster deployments. It integrates deeply with the Hadoop ecosystem and can give full play to the ability of large data tools.

We do not need to worry about ability analyze the bottleneck of complex bigdata systems, do not give up the classic OLAP theory, do not downgrade your service, do not worry about the business staff to unfamiliar large data tools. IndexR like MySQL, will be good for SQL.

after open sourcing IndexR, we have seen a lot of use cases, including the different teams at home and abroad. Interestingly, some teams are used in a special way, for example, to store a large number of complex detail data (single table design), or to do a detailed query of historical data. IndexR not only can be used in multidimensional analysis, business intelligence, such as the classic areas of OLAP, but also for Internet, public opinion monitoring, crowd behavior analysis and other emerging directions.

# # Contact Us

! [IndexR_icon] (IndexR_icon.png)

* Contact Email: IndexRdb@gmail. com
* QQ Discussion groups: 606666586 (IndexR discussion group)
