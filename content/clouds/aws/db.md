# AWS Databases

![Database Comparison](db-comprasion.png)

## Quick Comparison

| Database | Type | Use Case |
|----------|------|----------|
| **RDS** | Relational | Traditional SQL, complex queries, ACID compliance |
| **DynamoDB** | NoSQL Key-Value | High-speed data access, scalable apps, low latency |
| **ElastiCache** | In-Memory Cache | Session storage, caching, real-time leaderboards |
| **Redshift** | Data Warehouse | Analytics, OLAP, large-scale reporting |
| **Neptune** | Graph Database | Recommendation engines, social networks |
| **DocumentDB** | Document (MongoDB-like) | Document storage, flexible schemas |
| **Timestream** | Time Series | IoT metrics, application monitoring |
| **Keyspaces** | Wide-Column (Cassandra-like) | Time series data, high throughput |

## RDS (Relational Database Service)

**Best for:** Traditional applications with structured data

- Supports: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server
- Auto backups and failover
- Multi-AZ deployment for high availability
- Read replicas for scaling reads

## DynamoDB

**Best for:** Serverless applications needing fast, scalable NoSQL

- Fully managed, pay-per-request
- Millisecond latency at any scale
- Global tables for multi-region
- Query with partition key and sort key

## ElastiCache

**Best for:** Caching and session management

- Supports Redis and Memcached
- Sub-millisecond performance
- Cluster and replication support

## Redshift

**Best for:** Data warehousing and analytics

- Columnar storage optimized for analytics
- Handles petabyte-scale data
- Significantly cheaper than traditional data warehouses

## Neptune

**Best for:** Graph relationships and connections

- Social networks, recommendation engines
- Pattern detection, knowledge graphs
- Supports both Gremlin and SPARQL query languages

## DocumentDB

**Best for:** MongoDB-compatible document storage

- Fully managed, no infrastructure to maintain
- Automatic scaling
- Transaction support
