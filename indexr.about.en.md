## Introdution

IndexR is a distributed, relational database system based on HDFS, which focus on fast analyse, both for massive static(historical) data and rapidly ingesting realtime data. It is fully SQL supported, and designed for OLAP.

* **Fast Statistic Query** - IndexR store its data in columned, sorted, and highly compressed style. It provides effective, smart enough, so called rough set index. It could quickly target the relevant data and greatly reduce IO. It use the remarkable query engine [Drill](https://drill.apache.org/docs/performance/). IndexR is greatly suitable for ad-hoc OLAP.

* **Realtime Ingestion** - IndexR supports lightning fast realtime ingestion. As soon as events reach IndexR nodes, they are immediately queryable. Realtime data and historical data are accessed in the same query. No more lambda architecture, no more T+1. Unlike other similar system, IndexR will **NEVER** throw away any events, we don't explicitly require a column called *timestamp*. Currently IndexR supports realtime ingest events from [Kafka](http://kafka.apache.org/), more data sources will be supported soonly.

* **Hardware Efficiency** - IndexR is very hardware cost effective comparing to other similar systems/databases. It manually manage most of the memory(though it runs on JVM), with very tight structure. 

	> No more highly recommended SSD, huge RAM, high speed CPU, though they do speed up things anyway.

* **Highly Avaliable, Scalable, Manageable and Simple** - In modern world HA and Scalability is basic standard for distributed system. What we want to talk about is the Manageability and Simplicity. There is only one kind of IndexR node in the cluster, with very few required settings. Users can easily add/delete/update tables, dynamically update realtime ingestion settings. 

	> It is so simple and natural, just like using the nice classic Mysql.

* **Deep Integration with Hadoop Ecosystem** - IndexR stores its data on HDFS. It means you can directly run MapReduce job on the same data files without copying or transforming. You can manage your data with any hadoop tools. Just put them into the table directory, they are immediately queryable by IndexR. Normally we use [Hive](https://hive.apache.org/) to do the ETL of static data on HDFS and run offline analyse SQL. We are working on adapting IndexR with [Spark](http://spark.apache.org/)。
* **High Compression** - IndexR compress its data with highly rate, which can save you lots of disk space and reduce IO.

## Motivation

IndexR is inspired by many great projects, especially [Infobright](https://infobright.com/), [Pinot](https://github.com/linkedin/pinot) and [Druid](http://druid.io/). The idea of rough set index and the compression is original comes from Infobright's opensource version.

IndexR is original developed by [Sunteng](http://www.sunteng.com/), a leading internet company making bussiness around advertise marketing(DSP), website/mobile analytic and data management platform(DMP). Almost all of our products highly rely on a fast analyse system. It is difficult for us to find a suitable solution around existing products.

* Existing SQL on Hadoop is either too slow or without effective index. We need a faster query engine(Drill works pretty well) and an indexed file format with low disk cost.
* A k-v based system like [Storm](http://storm.apache.org/) + [Hbase](https://hbase.apache.org/), [Apache Phoenix](https://phoenix.apache.org/) or pre-build cube [Kylin](http://kylin.apache.org/) won't save us because it is too inflexible. There are too many dimensions and the query models is changing by weeks.
* Search engine systems, e.g. [Lucene](https://lucene.apache.org/) based the [ElasticSearch](https://www.elastic.co/) and [Solr](http://lucene.apache.org/solr/) are both support aggregations and sorting. The problem is their data modle is not cohesive enough, which cost too much RAM. Besides they offers poor performance on large data processing.
* [Druid](http://druid.io/) functionally meets most of our demands. The main problem is the [complex architecture](http://druid.io/docs/latest/design/design.html) detail exposed to user, which make it extremely difficult to maintain. Besides, Druid need to [mmap](http://druid.io/docs/latest/operations/rolling-updates.html) all files into memory before start up and query, in the case lots of files and without SSD, it could take more than half an hour to complete, and consume huge memory. Worse, the realtime ingestion of Druid could lost data, either caused by ingest task failed or delay timestamp events, which we could not tolerance.

	> In fact, we used to deploy a cluster of Druid in production, only after half a year of suffering we decided to build a better one.

## Performance

Hardware: 6 cores(24 threads) CPU, 60G RAM, HDD STATA with 7200 RPM.

* **Realtime ingestion speed** - maximum over 30K events / second / node / table. e.g. 10 nodes each serving 10 realtime tables can consumes 3M events within one second. We believe this is the best score ever around all similar systems.
* **Scan speed** - You may find much better performence in real production environment because IndexR can process multiple values at the same time with the help of modern CPU and processing platform(like Drill).
	* cold data: over 30 millions values / second / node.
	* hot data: over 100 millions values / second / node.
	* ~2.5 times as Parquet.
* **OLAP query** - In our production, we see 95% < 3s query latency on tables with 100 billions+ rows with 20 nodes.
	* 3~8 times as Parquet under the same Drill environment.
* **Compression** - In our production, we see average 10:1 compress rate comparing to raw csv format.
	* ~75% size of ORC file format. 

## Usage

* Supporting analytic dashboard, BI, etc.
* In areas like machine learning, time-series data collecting(monitoring, statistic).
* Quickly collect event logs into HDFS.