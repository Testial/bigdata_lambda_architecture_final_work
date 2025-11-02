# Big Data Final Project — Simple Lambda Architecture

**Repository:** Final work for Big Data course, MTS School of Data Analysts  
**Environment:** executed on the course YARN/Hadoop cluster using the supplied Jupyter notebook template.

## Project overview

This project implements a compact **Lambda-style data platform** that ingests ticket purchases in near-real time, enriches them with currency rates, computes streaming revenue metrics and produces several batch analytical reports from historical flight data. The solution demonstrates end-to-end ETL for both the **speed** (streaming) and **batch** layers plus a **serving** layer for fast lookups.

## What I implemented

* **Exploratory Data Analysis (EDA)** of Iceberg tables and Kafka topic; identified data quality/storage issues and proposed mitigations (example: local timestamps without timezone information affecting flight duration calculations).
* **Speed layer (near-real-time):**

  * Spark Structured Streaming to consume `tickets-topic` from **Apache Kafka**.
  * Enrichment with latest currency rates (Iceberg `currency` table) and normalization to USD.
  * Late-data handling: computed event lag, filtered out records with lag > 1 day.
  * Windowed aggregation (5-minute tumbling windows) to compute per-airline revenue; watermarking (1 minute) used to bound state.
  * Stream output persisted into **Apache Cassandra** (Cassandra table `airline_revenue`) using Spark Cassandra connector; checkpointing used to avoid duplicates.
* **Batch layer:**

  * Batch processing over Iceberg `flights` table to produce:

    * Route lists (departure/arrival cities).
    * Fleet reports (counts per model, total & average capacity for 2024–2025).
    * Top-5 airlines with lowest average flight duration (with assumptions noted).
    * Quarterly capacity reports with percent change using window functions.
  * Results saved back as Iceberg tables under user schema `ice_hdfs.student_user20.*`.
* **Serving layer:**

  * Data modeling in Cassandra: designed keyspaces/tables for serving routing, quarterly capacity and streaming revenue.
  * One-time batch replication from Iceberg ODS into Cassandra tables.
  * Example queries (via `cassandra-driver` and `cqlsh`) answering:

    * “Can airline X fly to city Y?”
    * “Can airline X carry N passengers based on last reported quarter?”
    * “What is airline X’s revenue in 2024?”
* **Operational considerations:** use of broadcast joins for small lookup tables, explicit spark session tuning (shuffle partitions, executor settings), use of Iceberg catalog configured via Hive metastore.

## Key technologies

* **Apache Spark**

  * Spark Structured Streaming (Kafka source, stateful aggregations, watermarks, windowing)
  * Spark DataFrame API for batch transformations, window functions, joins and saves
  * Spark on YARN; cluster execution via Jupyter notebook template
* **Apache Kafka**

  * Event ingestion (topic: `tickets-topic`) for the speed layer
* **Apache Iceberg (catalog via Hive)**

  * Table format for data lake (Iceberg catalog `ice_hdfs`), used to store `flights`, `currency` and derived tables
  * Partitioning and file format awareness (Parquet / ORC)
* **Apache Cassandra**

  * Serving layer for low-latency queries; data modeling oriented to partition/clustering keys for fast reads
  * `spark-cassandra-connector` for streaming & batch writes
* **Language / environment**

  * Python 3, PySpark, Jupyter Notebook
