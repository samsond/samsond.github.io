---
layout: post
title:  "Optimizing Queries in PostgreSQL: Partitioning and Indexing"
date:   2024-12-1 18:00:00 -0800
categories: database postgres 
ddescription: "An in-depth look at practical query optimization techniques in PostgreSQL, focusing on partitioning and indexing to improve query performance."
---

In this article, we will explore practical query optimization techniques in PostgreSQL using a real-world dataset of stock prices. We will focus on partitioning and indexing, and demonstrate their impact on query performance.

### Understanding the Dataset

For this article, we will use a stock price dataset from the GitHub repository [FNSPID_Financial_News_Dataset](https://github.com/Zdong104/FNSPID_Financial_News_Dataset). This dataset contains price data for various stocks over a period of time.

### Schema of the stock_prices Table
Here is the schema of the **stock_prices** table that we are working on:

```sql
CREATE TABLE stock_prices (
    stock_id VARCHAR,
    date DATE,
    open NUMERIC,
    high NUMERIC,
    low NUMERIC,
    close NUMERIC,
    adj_close NUMERIC,
    volume NUMERIC,
    sentiment_gpt NUMERIC,
    news_flag NUMERIC,
    scaled_sentiment NUMERIC
);

```

### Partitioning
Partitioning involves dividing a large table into smaller, more manageable pieces called partitions. This can improve query performance by allowing the database engine to scan only the relevant partitions.

### Before Partitioning
Let's run a query to fetch data for a specific date range and analyze its performance without partitioning.

```sql
EXPLAIN ANALYZE
SELECT * FROM stock_prices WHERE date >= '2023-01-01';

                                                    QUERY PLAN                                                     
-------------------------------------------------------------------------------------------------------------------
 Seq Scan on stock_prices  (cost=0.00..5875.21 rows=11644 width=96) (actual time=0.051..17.036 rows=11772 loops=1)
   Filter: (date >= '2023-01-01'::date)
   Rows Removed by Filter: 116165
 Planning Time: 0.157 ms
 Execution Time: 17.522 ms
(5 rows)
```

### Implementing Partitioning

The results clearly demonstrate the inefficiency of scanning the entire table for queries targeting specific date ranges. Let's explore how partitioning can dramatically improve performance.

First, let's create a table named stock_prices_partitioned to enable partitioning:

```sql
CREATE TABLE stock_prices_partitioned (
    stock_id VARCHAR,
    date DATE,
    open NUMERIC,
    high NUMERIC,
    low NUMERIC,
    close NUMERIC,
    adj_close NUMERIC,
    volume NUMERIC,
    sentiment_gpt NUMERIC,
    news_flag NUMERIC,
    scaled_sentiment NUMERIC
) PARTITION BY RANGE (date);
```

Use the script below to dynamically create partitions for each year.

```bash
DATABASE="stock_data"
USER="sami"

for year in {2009..2023}; do
    next_year=$((year + 1))
    psql -U $USER -d $DATABASE -c "
    CREATE TABLE stock_prices_${year} PARTITION OF stock_prices_partitioned
    FOR VALUES FROM ('$year-01-01') TO ('$next_year-01-01');"
done

```

### Migrating Data to Partitioned Table

We will migrate the data from the original table to the partitioned table and verify the data count to ensure the migration was successful.


```sql
INSERT INTO stock_prices_partitioned
SELECT * FROM stock_prices;
```

### Check Partition Distribution

To verify the distribution of data across partitions, we will run a query to count the number of rows in each partition. This helps us understand how the data is distributed and ensures that the partitions are being used correctly.


```sql
SELECT tableoid::regclass AS partition_name, COUNT(*) 
FROM stock_prices_partitioned 
GROUP BY tableoid;

  partition_name   | count 
-------------------+-------
 stock_prices_2009 |   970
 stock_prices_2010 |  6184
 stock_prices_2011 |  6552
 stock_prices_2012 |  6675
 stock_prices_2013 |  7524
 stock_prices_2014 |  7633
 stock_prices_2015 |  8469
 stock_prices_2016 |  9075
 stock_prices_2017 |  9542
 stock_prices_2018 |  9888
 stock_prices_2019 | 10700
 stock_prices_2020 | 10753
 stock_prices_2021 | 10680
 stock_prices_2022 | 11520
 stock_prices_2023 | 11772
(15 rows)

```


### After Partitioning

Run the same query again and analyze its performance with partitioning

```bash
stock_data=# EXPLAIN ANALYZE
SELECT * FROM stock_prices_partitioned WHERE date >= '2023-01-01';
                                                                  QUERY PLAN                                                                   
-----------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on stock_prices_2023 stock_prices_partitioned  (cost=0.00..343.15 rows=11771 width=97) (actual time=0.057..2.595 rows=11772 loops=1)
   Filter: (date >= '2023-01-01'::date)
 Planning Time: 0.862 ms
 Execution Time: 3.200 ms
(4 rows)


```

Using partitioning significantly reduces execution time, improving performance from **17.522 ms** to **3.200 ms** achieving roughly **81%** reduction in execution time.

### Indexing
Indexing allows the database engine to locate data quickly without scanning the entire table. Since stock_id is already a primary key, it is indexed by default. Let's create an additional index on the date column to optimize queries that filter by date.

### Before Indexing
Let's run a query to fetch data for a specific date and analyze its performance without indexing.

```sql
stock_data=# EXPLAIN ANALYZE
SELECT * FROM stock_prices WHERE stock_id = 'AAPL';
                                                  QUERY PLAN                                                   
---------------------------------------------------------------------------------------------------------------
 Seq Scan on stock_prices  (cost=0.00..5875.21 rows=409 width=96) (actual time=0.040..23.852 rows=387 loops=1)
   Filter: ((stock_id)::text = 'AAPL'::text)
   Rows Removed by Filter: 127550
 Planning Time: 0.692 ms
 Execution Time: 23.907 ms
(5 rows)
```

### Adding an Index
The reason for the above query's inefficiency is that it performs a full table scan, which is a costly operation. To address this, let's create an index on the stock_id column to optimize the query.

```sql
CREATE INDEX idx_stock_id ON stock_prices(stock_id);
```

### After Indexing
Run the same query again and analyze its performance with the index.

```sql
stock_data=# EXPLAIN ANALYZE                                     
SELECT * FROM stock_prices WHERE stock_id = 'AAPL';
                                                           QUERY PLAN                                                            
---------------------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_stock_id on stock_prices  (cost=0.29..28.45 rows=409 width=96) (actual time=0.052..0.117 rows=387 loops=1)
   Index Cond: ((stock_id)::text = 'AAPL'::text)
 Planning Time: 0.539 ms
 Execution Time: 0.164 ms
(4 rows)
```

Using an index significantly improves query performance, reducing execution time from **23.907 ms** (sequential scan) to **0.164 ms** (index scan), achieving a **99.31%** reduction in execution time.


### Conclusion
In this article, we explored practical query optimization techniques in PostgreSQL using a real-world dataset of stock prices. We focused on partitioning and indexing, and demonstrated their impact on query performance. By implementing these strategies, you can significantly improve the performance of your queries, reducing both query speed and cost.

This article was inspired by the dataset from the GitHub repository [FNSPID_Financial_News_Dataset](https://github.com/Zdong104/FNSPID_Financial_News_Dataset).

Happy querying!


