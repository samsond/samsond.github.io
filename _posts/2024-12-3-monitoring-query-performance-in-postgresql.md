---
layout: post
title:  "Monitoring Query Performance in PostgreSQL"
date:   2025-12-3 18:00:00 -0800
categories: observability postgres 
description: "A practical guide to leveraging Prometheus for monitoring and improving PostgreSQL query performance."
---

This article explores how Prometheus can be used to monitor PostgreSQL query performance and guide dynamic query optimization. By analyzing key database metrics, such as query execution times, index usage, and cache hit ratios, we demonstrate practical optimization techniques that improve query performance in PostgreSQL using a real-world stock price dataset.

In our previous article, we focused on partitioning and indexing as static optimization techniques to improve query performance in PostgreSQL. These techniques help reduce query execution times by organizing data more efficiently and enabling faster data retrieval.

While static optimization techniques are essential, continuous optimization based on real-time metrics is crucial for maintaining optimal query performance. Monitoring query performance allows us to identify and address performance issues as they arise, ensuring that our database remains efficient and responsive.

### Prometheus

Prometheus is an open-source, metrics-based monitoring system designed for reliability and scalability. It collects and stores metrics as time series data, providing powerful querying capabilities and alerting features.

Prometheus provides actionable insights into query performance by collecting detailed metrics on query execution times, index usage, cache hit ratios, and more. These insights enable database administrators and developers to make informed decisions and implement dynamic optimization techniques.

### Setting Up Prometheus for PostgreSQL
We need to follow these steps:

* Install and Configure Prometheus
* Install and configure PostgreSQL exporter

### Install and Configure Prometheus
* Download and install Prometheus from the [official website](https://prometheus.io/download/).


### Configure Prometheus 

Create a configuration file **prometheus.yml** with the following content:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'postgresql'
    static_configs:
      - targets: ['localhost:9187']
```

### Run Prometheus 

Start Prometheus with the configuration file:
```bash
./prometheus --config.file=prometheus.yml
```

### Verify Prometheus 

Head over to **http://localhost:9090** to access the Prometheus web interface. 

### Install and Configure PostgreSQL Exporter
The PostgreSQL Exporter is a Prometheus exporter for PostgreSQL metrics. It collects and exposes metrics from PostgreSQL databases, allowing Prometheus to scrape and store these metrics for monitoring and analysis.

* Download and install the PostgreSQL Exporter from the [official repository](https://github.com/prometheus-community/postgres_exporter).

> **Side note** If you are working on MacOs you would be unable to run due to Remove 
macOS adds a quarantine attribute to files downloaded from the internet. You need to remove this attribute with the following:
```bash 
xattr -d com.apple.quarantine ./postgres_exporter
```



* Configure PostgreSQL Exporter
Let's export essential environment variables to configure the PostgreSQL Exporter. Set the following environment variables:

```bash
export DATA_SOURCE_URI="localhost:5432/stock_data?sslmode=disable"
export DATA_SOURCE_USER="db_user_name"
export DATA_SOURCE_PASS="YourPassword"
```

### Run PostgreSQL Exporter

Start the PostgreSQL exporter as follows:

```bash
./postgres_exporter
```

### Verify PostgreSQL Exporter

Head over to **http://localhost:9187/metrics** to see if you can see your database metrics listed. 

### Query Execution Times
Monitor query execution times to identify slow queries and optimize them. Prometheus collects metrics on query durations, allowing you to pinpoint queries that need optimization.

Here are the practical steps to achieve this.

#### Add pg_stat_statements

According to the [PostgresSQL](https://www.postgresql.org/docs/current/pgstatstatements.html) documentation **pg_stat_statements** module helps to track planning and execution statistics of all SQL statements executed by a server.

To use it, we must add `pg_stat_statements` to `shared_preload_libraries` in `postgresql.conf`, and this requires a server restart due to its need for extra shared memory.

```bash
shared_preload_libraries = 'pg_stat_statements' 
```

> **Side Note**: To better help in locating the file where **postgresql.conf** resides log into PostgresSQL
and run the following query
```sql
stock_data=# show config_file;
                   config_file                   
-------------------------------------------------
 /opt/homebrew/var/postgresql@15/postgresql.conf
(1 row)
```

Additionally query identifier calculation must be enabled and it must be either auto or on as shown below:

```sql
stock_data=# show compute_query_id;
 compute_query_id 
------------------
 auto
(1 row)
```

#### Restart PostgreSQL
```bash
# On Mac using brew
brew services restart postgresql@15
# On Linux
sudo systemctl restart postgresql
```

#### Create the pg_stat_statements Extension

Connect to your PostgreSQL database and create the extension:

```sql
CREATE EXTENSION pg_stat_statements;
```

#### Re-Run PostgreSQL Exporter: 

Start the PostgreSQL Exporter again with pg_stat_statements collector flags as follows

```bash
./postgres_exporter --collector.stat_statements
```

#### Verify Metrics in Prometheus

Look for metrics like the following

```text
pg_stat_statements_calls_total Number of times executed

pg_stat_statements_rows_total Total number of rows retrieved or affected by the statement

pg_stat_statements_block_read_seconds_total Total time the statement spent reading blocks, in seconds

```

For this metrics **pg_stat_statements_rows_total** you might get similar to the below
```

pg_stat_statements_rows_total{datname="stock_data", instance="localhost:9187", job="postgresql", queryid="2569472049308808451", user="sami"}
774
```

To really know which sql statement for a given queryid use the below query

```sql
SELECT queryid, query
FROM pg_stat_statements
WHERE queryid = '2569472049308808451';
```

we can identify the top 5 queries that have processed the highest number of rows. This insight helps us understand which queries are handling large datasets.

```text
topk(5, pg_stat_statements_rows_total)
```

Sample output from PromQL query
```
pg_stat_statements_rows_total{datname="stock_data", instance="localhost:9187", job="postgresql", queryid="-6680645881763619241", user="sami"}76104
pg_stat_statements_rows_total{datname="stock_data", instance="localhost:9187", job="postgresql", queryid="-3533477022277510685", user="sami"}
10908
pg_stat_statements_rows_total{datname="stock_data", instance="localhost:9187", job="postgresql", queryid="3048661863768995886", user="sami"}
7315
pg_stat_statements_rows_total{datname="stock_data", instance="localhost:9187", job="postgresql", queryid="7153408741141088141", user="sami"}
5762
pg_stat_statements_rows_total{datname="stock_data", instance="localhost:9187", job="postgresql", queryid="8227976836737990298", user="sami"}
5134
```

### Conclusion
In this article, we provided practical steps for monitoring query execution times using Prometheus and PostgreSQL Exporter. By enabling the pg_stat_statements extension and querying execution time metrics in Prometheus, you can identify slow queries and take steps to optimize them.