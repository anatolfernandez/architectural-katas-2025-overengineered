# ADR - Two-Tier Caching Strategy for Customer Risk Assessment AI

## Status
**Accepted**

## Context
The Customer Risk Assessment AI (CRAI) predicts driver risk scores to calculate dynamic pricing multipliers (0.8x - 1.5x). The pricing API requires a response within 120ms end-to-end, with the CRAI inference budgeted at ~50ms maximum.

Even with features served from Redis Online Feature Store (< 5ms latency), running XGBoost model inference adds significant latency:
- Feature retrieval from Redis: ~5ms
- Model loading (if not cached): ~10-20ms
- Model inference (XGBoost on 60 features): ~15-25ms
- **Total: 30-50ms per prediction**

For customers who book multiple times per day, we repeatedly compute the same risk score. With 500K active customers and 100K booking price requests per day, this represents significant wasted compute and latency.

Additionally, customer risk profiles change slowly (daily, not hourly), so real-time inference on every request is unnecessary.

## Decision Drivers
- **Latency requirement**: CRAI must return results in < 50ms to meet overall pricing API SLA of < 120ms
- **Compute efficiency**: Reduce redundant model inference for the same customer
- **Cost**: XGBoost inference costs ~0.5ms CPU time per prediction; caching saves ~$500/month in compute
- **Risk profile stability**: Customer driving behavior changes slowly (daily patterns, not hourly)
- **Cache freshness**: Daily updates acceptable for risk assessment (risk doesn't change intra-day)
- **Availability**: Must handle cache misses gracefully (fallback to real-time inference)

## Options Considered

### Option 1: No Caching - Real-time Inference Only
Run XGBoost inference on every request, fetching features from Redis.

**Pros:**
- Always fresh predictions
- Simple architecture
- No stale data risk

**Cons:**
- **Higher latency**: 30-50ms per request (tight margin for SLA)
- **Wasted compute**: Repeated inference for same customer
- **Higher cost**: ~$500/month additional compute
- **Scalability concerns**: CPU bottleneck during peak loads

### Option 2: Single-Tier Caching - Cache in Online Feature Store
Store pre-computed risk scores alongside features in the same Redis instance (Feast Online Store).

**Pros:**
- Simple integration with Feast
- One caching layer to manage
- Lower latency than no caching

**Cons:**
- **Inflexible refresh rate**: Tied to feature materialization schedule (nightly)
- **Namespace collision**: Risk scores mixed with features (confusing semantics)
- **No independent TTL**: Cannot set different expiry for predictions vs. features
- **Feast overhead**: Feast adds serialization/deserialization overhead (~2ms)

### Option 3: Two-Tier Caching - Dedicated CRAI Cache + Feature Store Fallback ✅
Implement a dedicated Redis namespace for CRAI predictions with fallback to feature store + real-time inference.

**Tier 1 - CRAI Cache** (Redis namespace `crai_cache:*`):
- Stores final risk scores (not features)
- TTL: 24 hours
- Updated via nightly batch job (pre-compute for all active customers)
- Ultra-fast lookup: ~2-3ms

**Tier 2 - Real-time Inference** (fallback on cache miss):
- Fetch features from Online Feature Store
- Load model (cached in memory)
- Run XGBoost inference
- Cache result in Tier 1 for 24 hours

**Pros:**
- **Optimal latency**: 95% of requests in < 10ms (cache hit)
- **Cost-effective**: Reduces inference compute by 95%
- **Independent control**: Can adjust TTL and refresh rate separately from features
- **Graceful degradation**: Falls back to real-time inference on cache miss
- **Scalable**: Pre-computation happens offline (doesn't impact API load)

**Cons:**
- **Increased complexity**: Need to manage two caching layers
- **Stale predictions**: Up to 24 hours old (acceptable for risk assessment)
- **Batch job dependency**: If nightly job fails, cache becomes stale

## Decision
**We will implement a two-tier caching strategy** (Option 3) with the following design:

### Tier 1: CRAI Cache (Primary)
- **Storage**: Redis namespace `crai_cache:customer_id:{customer_id}`
- **Data structure**:
  ```json
  {
    "customer_id": "CUST_12345",
    "risk_score": 0.73,
    "risk_multiplier": 1.2,
    "model_version": "v5",
    "computed_at": "2025-10-23T02:15:00Z",
    "expires_at": "2025-10-24T02:15:00Z"
  }
  ```
- **TTL**: 24 hours
- **Population**: Nightly batch job (2 AM) pre-computes risk for all active customers (defined as: booked in last 90 days)
- **Batch job process**:
  1. Query active customers from Data Lakehouse
  2. Fetch features from Online Feature Store (batch operation)
  3. Load CRAI model from MLflow
  4. Run batch inference (XGBoost on 500K customers in parallel on Databricks)
  5. Write predictions to CRAI cache
  6. **Duration**: ~30 minutes for 500K customers

### Tier 2: Real-time Inference (Fallback)
- **Trigger**: Cache miss (customer not in CRAI cache)
- **Process**:
  1. Log cache miss event (for monitoring)
  2. Fetch features from Online Feature Store (Redis Feast)
  3. Load model from MLflow (cached in API memory)
  4. Run XGBoost inference
  5. Write result to CRAI cache (TTL: 24 hours)
  6. Return prediction
- **Expected latency**: 50-80ms
- **Expected frequency**: 5% of requests (new customers, expired cache entries)

### API Logic
```python
def get_customer_risk(customer_id: str) -> RiskScore:
    # Tier 1: Check CRAI cache
    cached = redis.get(f"crai_cache:customer_id:{customer_id}")
    if cached:
        metrics.increment("crai.cache.hit")
        return parse_risk_score(cached)
    
    # Tier 2: Real-time inference
    metrics.increment("crai.cache.miss")
    features = feast_store.get_online_features(
        features=["customer_behavior_features:*"],
        entity_rows=[{"customer_id": customer_id}]
    )
    risk_score = model.predict(features)
    
    # Cache result
    redis.setex(
        f"crai_cache:customer_id:{customer_id}",
        86400,  # 24 hours
        serialize_risk_score(risk_score)
    )
    
    return risk_score
```

## Consequences

### Positive
- **Dramatic latency improvement**: 95% of requests served in < 10ms (vs. 30-50ms without caching)
- **Cost savings**: Reduce XGBoost inference by 95% = ~$500/month compute savings
- **Better p99 latency**: Eliminates inference latency spikes during peak load
- **Scalability**: Pre-computation offloaded to batch job (doesn't impact API)
- **Graceful degradation**: Fallback to real-time inference ensures no failures on cache miss

### Negative
- **Stale predictions**: Risk scores up to 24 hours old (acceptable trade-off for this use case)
- **Increased operational complexity**: Need to monitor batch job health and cache freshness
- **Additional memory**: CRAI cache requires ~50GB Redis memory (500K customers × 100KB per entry)
- **Cold start latency**: New/inactive customers experience 50-80ms latency on first request

### Neutral
- **Batch job dependency**: System health depends on nightly batch job success (mitigated with alerting)
- **Cache warming**: New customers not pre-computed; warmed on first request
- **TTL tuning**: May need to adjust 24-hour TTL based on risk profile change frequency

