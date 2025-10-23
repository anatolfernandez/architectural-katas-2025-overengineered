# ADR - Separate Offline and Online Feature Stores

## Status
**Accepted**

## Context
Our ML system needs to serve features for two distinct purposes:
1. **Training**: Generate historical feature datasets with point-in-time correctness to train ML models (Customer Risk Assessment AI and Demand Prediction AI)
2. **Inference**: Serve features in real-time during prediction requests with sub-10ms latency to meet the < 120ms end-to-end pricing API SLA

The challenge is that these two use cases have conflicting requirements:
- Training requires access to large-scale historical data (TBs), complex aggregations (window functions, joins), and point-in-time correctness to prevent data leakage
- Inference requires ultra-low latency, high throughput (10k+ req/sec), and simple key-value lookups

A single storage system cannot efficiently satisfy both requirements.

## Decision Drivers
- **Latency requirements**: Inference must complete in < 10ms for feature retrieval; training latency is less critical (minutes to hours acceptable)
- **Data volume**: Need to store 2+ years of historical telemetry data (TBs) for training; only need latest feature values (GB) for inference
- **Query complexity**: Training requires complex SQL queries with window functions; inference needs simple key lookups by entity ID
- **Cost**: Storing full history in a low-latency store (Redis) would be prohibitively expensive
- **Point-in-time correctness**: Training data must only use features available at the time of prediction to avoid data leakage
- **Scalability**: System must handle 500K customers with 100K predictions per day

## Options Considered

### Option 1: Single Store - Databricks Only
Use Databricks for both training and inference, querying Gold tables directly via JDBC/REST API.

**Pros:**
- Simpler architecture (one data store)
- No synchronization needed
- Always fresh data
- Supports complex queries

**Cons:**
- **High latency**: 100-500ms per query (unacceptable for real-time inference)
- **Limited throughput**: Cannot handle 10k+ req/sec
- **Expensive**: High compute cost for serving simple lookups

### Option 2: Single Store - Redis Only
Use Redis as the sole feature store, loading all historical data.

**Pros:**
- Ultra-low latency (< 5ms)
- High throughput (10k+ req/sec)
- Simple architecture

**Cons:**
- **Memory constraints**: Storing TBs of historical data in Redis is cost-prohibitive (~$10K+/month)
- **No complex queries**: Cannot perform window functions, joins, or point-in-time corrections
- **Limited historical depth**: Can only store recent data (30-90 days max)
- **Not suitable for training**: Cannot generate proper training datasets

### Option 3: Separate Offline (Databricks) and Online (Redis) Stores with Materialization ✅
Use Databricks for training and Redis for inference, with periodic materialization jobs to sync latest features.

**Pros:**
- **Optimal for training**: Databricks handles large-scale data, complex queries, point-in-time joins
- **Optimal for inference**: Redis provides < 5ms latency and high throughput
- **Cost-effective**: Only store latest features in Redis (GB scale), full history in Databricks
- **Scalable**: Each store optimized for its use case

**Cons:**
- **Increased complexity**: Need to manage two stores and synchronization
- **Eventual consistency**: Online store may be slightly stale (acceptable for our use case)
- **Operational overhead**: Need to monitor materialization jobs

## Decision
**We will use separate offline and online feature stores** (Option 3) with the following architecture:

### Offline Feature Store
- **Technology**: Feast connected to Databricks (Delta Lake)
- **Purpose**: Training data generation, feature registry, historical feature storage
- **Data retention**: 2+ years of historical features
- **Query patterns**: Complex SQL with window functions, point-in-time joins
- **Access pattern**: Batch queries during model training (weekly)

### Online Feature Store
- **Technology**: Redis Cluster (3+ nodes)
- **Purpose**: Real-time feature serving for inference
- **Data retention**: Latest feature values only (90-day TTL)
- **Query patterns**: Key-value lookups by entity ID (customer_id, parking_bay_id)
- **Access pattern**: 100K+ req/day with < 5ms latency

### Materialization Process
- **Frequency**: Nightly batch job (2 AM) via Databricks scheduled jobs
- **Process**: Feast reads latest features from offline store → writes to Redis
- **Strategy**: Incremental updates (only changed features)
- **Monitoring**: Track materialization lag, success rate, and data freshness

## Consequences

### Positive
- **Meets latency requirements**: < 5ms feature retrieval for inference (vs. 100-500ms with Databricks-only)
- **Cost-effective**: Redis memory footprint ~100GB (500K customers × 200KB features) = ~$500/month vs. ~$10K+/month for full history
- **Scalable**: Can handle 10k+ req/sec for inference; Databricks scales for training
- **Point-in-time correctness**: Feast + Databricks ensures training data has no leakage
- **Flexibility**: Can evolve each store independently (e.g., migrate Redis → DynamoDB without affecting training)

### Negative
- **Operational complexity**: Need to monitor and maintain two data stores
- **Eventual consistency**: Online features may be up to 24 hours stale (mitigated by nightly materialization)
- **Synchronization risk**: If materialization fails, online store becomes stale (mitigated by alerting)
- **Data duplication**: Latest features stored in both Databricks (Gold tables) and Redis

### Neutral
- **Increased deployment complexity**: Need to provision Redis cluster, configure Feast, set up materialization jobs
- **Learning curve**: Team needs to understand Feast framework and Redis operations
