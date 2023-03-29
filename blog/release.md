# Launching Quokka: Distributed Polars on Ray
**Apr-01-2023**

## What is Quokka?

Today I am happy to announce that after more than a year and half since the conception of the project, Quokka is ready for its first Alpha release, version 0.2.0.

I posted a [progress report](https://github.com/marsupialtail/quokka/blob/master/blog/why.md) on Quokka at the end of last year, and was surprised and honored to make it to Hacker News top ranked for a day. 

Some of the motivations for why I built Quokka is described in the [progress report](https://github.com/marsupialtail/quokka/blob/master/blog/why.md). Quick recap: it is my belief that the data world is (or should be) moving away from JVM-based legacy tools to Python-native stacks on top of Rust or C. The Python + Rust/C stack offers both the ease-of-use of an interpreted language that executes line by line and no-GC, no-JIT, no-JVM, no-BS bare metal performance.

On single machine, most tools are now Python + Rust/C: Pandas, Polars, DuckDB, NumPy, Tensorflow, PyTorch... It truly pains my heart that we are stuck with things like SparkSQL on JVM once we need a distributed data solution. It pained the creators of SparkSQL too, who came up with the impressive Photon engine. Unfortunately, it is not open source and out of reach for people who aren't using Databricks. 

There have been prior efforts to bring the PyData stack to distributed settings. [Modin](https://github.com/modin-project/modin) and [Dask](https://github.com/dask/dask) are the best known examples. I have used them both in my previous jobs and personal projects, but have found their performance to be unsatisfactory compared to SparkSQL (>5x slower) on queries that do more than apply an embarassingly parallel UDF, especially on queries that involve heavy data shuffles, aka joins.

So naturally I had to write my own framework, Quokka, based on [Polars](https://github.com/pola-rs/polars), [DuckDB](https://github.com/duckdb/duckdb), [Arrow Flight](https://github.com/apache/arrow) and [Ray](https://github.com/ray-project/ray). Quokka currently supports a DataFrame API syntax heavily inspired by Polars, and limited SQL expressions. Here is its performance on the TPC-H benchmark at SF100 with Parquet input data on four r6id.2xlarge machines. 

<p align="center">
  <img src="https://github.com/marsupialtail/quokka/blob/master/docs/docs/tpch-parquet.svg?raw=true" alt="Title"/>
</p>

At least on this benchmark (less queries 20, and 21 which OOMs), Quokka achieves what I set out to do: a Python + Rust/C framework that beats SparkSQL on join-heavy SQL workloads. The Quokka code for these queries can be found [here](https://github.com/marsupialtail/quokka/blob/master/apps/tpc-h/tpch.py). 

## Why is Quokka fast?

**Firstly, like Polars and unlike Modin, Quokka intentionally deviates from the Pandas API**. Judging from the ChatGPT-like rise in popularity of [Polars](https://github.com/pola-rs/polars/stargazers), I do not think people are that attached to the Pandas API itself. If there is a clean and performant alternative, people will happily switch (like I did). The key culprit for Pandas' poor performance is lazy (or eager) execution. This precludes any kind of sophisticated optimizations that could drastically reduce the amount of IO or compute. For example, if I type `a = pd.read_csv("test.csv")`, Pandas will read in all of the CSV file into memory, even if you are only going to use the first ten rows.

Any kind of performance-oriented DataFrame library has to take a lazy approach, like Polars LazyFrame and PySpark DataFrames. The Quokka API tries to emulate the Polars API as much as possible. If you are already a Polars user, switching to the Quokka API for distributed workloads should feel seamless. While Polars still let's you use eager semantics, that is heavily discouraged in Quokka.

I don't claim to offer 100% of the functionality of Polars (yet), but I think the most important things should be there: `filter`, `select`, `join`, `union`, `with_columns` etc. I also offer SQL analogs of these functions such as `filter_sql` where the user can supply a SQL predicate directly, like `d.filter_sql("l_orderkey > 10 and l_tax between 0.1 and 0.3")`. 

**Adopting a lazy API allows Quokka to adopt optimizations such as predicate pushdown, early projection, and even cardinality-based join ordering**. These are the bread-and-butter optimizations inside an OLAP query engine like Spark's Catalyst, now available in a DataFrame library. I thought about using Spark's Catalyst or Apache Calcite, but decided to write everything myself with the help of [SQLGlot](https://github.com/tobymao/sqlglot). If the whole point is to cut the JVM, then we are going to cut the JVM. Brief description of what these optimizations are:
- **Predicate pushdown** breaks down filters and "pushes" them down as far as possible. For example, if you filter on the join result of two tables A and B, you can break down your filter into parts that apply to either A or B, filter them first, and join the filtered result. This reduces the amount of work for the join, and in some case can even reduce the amount of data to be read from disk or network.
- **Early projection**: if you are only ever going to use a subset of a table's columns, then we are only going to read that subset of columns. Greatly helps when reading from columnar data formats like Parquet that stores data by column.
- **Cardinality-based join ordering**: most serious SQL engines (and Quokka) now employ this optimization. When you are joining three tables A.id = B.id and B.id2 = C.id2, Quokka needs to decide if it should join A to B first and then the result to C, or join B to C first and then the result to A. This depends on which intermediate result is larger. Of course this depends on the data distribution, the filters pushed down to A and B, and the sizes of the inputs. Quokka estimates all of these by sampling the input data. 

**Cool. But SparkSQL has all these. Is Quokka doing something above and beyond?**   It is. SparkSQL is famously known for its stage-at-a-time execution paradigm. A stage, such as reading Parquets into RDD, or shuffling RDDs to perform a join, executes completely before the next stage can begin. This simplifies fault tolerance and abides to the MapReduce programming model, facilitating adoption back when it was introduced a decade ago.

However, most analytical SQL query engines in the past decade have shifted towards a push-based pipelined approach where batches of data are pushed through a deep pipeline. This: 
- Enables pipeline parallelism between many stages, overlapping IO and compute
- Avoids the need to materialize potentially immense intermediate datasets in memory/disk. 

If you'd like to learn more about this way of executing SQL queries, I'd encourage you to read more about DuckDB's [philosphy](https://db.in.tum.de/~leis/papers/morsels.pdf). Quokka subscribes to this execution model, and by default executes many stages in parallel in a deep pipeline. 

Of course, Quokka also benefits from Polars and DuckDB's heavily optimized Rust and C++ kernels for aggregations, filtering and joining which make heavy use of SIMD and cache optimizations. This makes Quokka up to 3x faster on aggregation-only queries such as TPC-H 1.

A key novelty of Quokka is that it maintains the fault tolerance properties of Spark in this pipelined execution model. Quokka does so through the notion of dynamic lineage. In Spark, 

## How to use Quokka?

Quokka can be easily installed: `pip3 install pyquokka`. It requires `polars>=0.16.15` and `ray>=2.0.0`. While I can not guarantee that Quokka does not have bugs today, I can guarantee that I will pay you money if you find [bugs](https://marsupialtail.github.io/quokka/). Since I am a poor grad student, unfortunately I can only fund up to $2000. 

Quokka is heavily integrated with Ray. You can deploy it by setting up a dedicated Ray cluster, running it on an existing Ray cluster, or locally on your laptop. The local option is more for testing and development. For production single-machine workloads, you should just use Polars or DuckDB. 

You can use Quokka as a way to slice and dice a TB-scale remote dataset on S3 into a manageable size to load into a Polars DataFrame locally for further analysis, or use it as a ETL tool to pipe data into Ray Data or Ray Train for downstream machine learning training. Quokka is especially helpful for those of you who do not want to run both a Spark cluster and a Ray cluster in your ML workloads.

<p align="center">
  <img src="https://github.com/marsupialtail/quokka/blob/master/docs/docs/intended_use.svg?raw=true" alt="Title"/>
</p>

You can explore Quokka's current API in the [tutorial](https://marsupialtail.github.io/quokka/simple/) or the API references. If you find it lacking, please join the Quokka [Discord](https://discord.gg/6ujVV9HAg3) where you can bug me constantly for new features. Here is a quick code snippet that reads a few tables from Parquet files in S3, join them and produces a Ray Dataset: 
```
# assume we have initialized EC2 cluster, check API reference for QuokkaClusterManager
>>> qc = QuokkaContext(cluster)
>>> lineitem = qc.read_parquet(s3_path_parquet + "lineitem.parquet/*")
>>> orders = qc.read_parquet(s3_path_parquet + "orders.parquet/*")
# filter with dataframe syntax
>>> d = lineitem.filter(lineitem["l_commitdate"] < lineitem["l_receiptdate"])
>>> d = orders.join(d, left_on="o_orderkey", right_on="l_orderkey", how = "semi")
# filter with SQL syntax
>>> d = d.filter_sql("o_orderdate >= date '1993-07-01'")
>>> d = d.select(["o_orderkey", "o_custkey", "o_orderstatus", "o_totalprice", "o_orderdate", "o_orderpriority", "o_clerk", "o_shippriority"])
>>> d.explain() # this will produce a PDF execution graph you can visualize 
>>> d = d.compute()
>>> dataset = d.to_ray_dataset()
```

Quokka is meant to complement Ray Data. As a result I am not going to prioritize implementing things that Ray Data already does very well, like sorting or random shuffling. I will focus more on improving the performance and stability of the terabyte-scale joins that typically precede such operations.

Quokka is currently mostly fault tolerant. It will be fully fault tolerant when Ray allows programmatic controls of object replication in its object store. Without getting into more details, let's just say you should not rely on its fault tolerance mechanism in production. If this is an issue for you, I'd love to [hear]((https://discord.gg/6ujVV9HAg3) ) from you.

## Credits
[Apache Arrow](https://github.com/apache/arrow), [Ray](https://github.com/ray-project/ray), [Polars](https://github.com/pola-rs/polars), [DuckDB](https://github.com/duckdb/duckdb), [SQLGlot](https://github.com/tobymao/sqlglot), [Redis](https://github.com/redis/redis). 

Shoutouts: [Alex Aiken](https://www.linkedin.com/in/alex-aiken-575aa7b7/), [Tim Tully](https://www.linkedin.com/in/timothytully/), [Toby Mao](https://www.linkedin.com/in/toby-mao/), [Peter Kraft](https://www.linkedin.com/in/peter-kraft-110318b0/), [Qian Li](https://www.linkedin.com/in/qianli15/), [Sarah Fujimori](https://www.linkedin.com/in/sarah-fujimori/), [Emanuel Adamiak](https://www.linkedin.com/in/emanuel-adamiak-699a27236/), [Fabio Ibanez](https://www.linkedin.com/in/fabioibanez/). 