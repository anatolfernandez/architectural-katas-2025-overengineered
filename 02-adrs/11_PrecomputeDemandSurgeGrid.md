# ADR - Pre-compute Demand Surge Grid for Real-time Pricing

## Status
**Accepted** - 2025-10-23

## Context
The Demand Prediction AI adjusts vehicle pricing based on predicted demand at parking bays, allowing the platform to:
1. **Incentivize desired parking behavior**: Lower prices for high-supply bays, higher prices for high-demand bays
2. **Optimize fleet distribution**: Guide customers to park where vehicles are needed
3. **Maximize revenue**: Capture willingness-to-pay during high-demand periods

The pricing API must return personalized prices within 120ms. Demand prediction needs to factor in:
- **Real-time event detection**: Concerts, sports games, public transport disruptions within 2km of destination bay
- **Weather forecast integration**: Rainy weather reduces scooter demand, increases car demand
- **Time-series patterns**: Rush hour, day of week, historical demand trends
- **Complex ML models**: LightGBM or LSTM models with 40+ features

Running a full ML inference pipeline on every pricing request would add 100-200ms latency, exceeding the SLA.

Additionally, demand predictions are **context-dependent but not customer-specific**: The same parking bay at the same time has the same surge factor for all customers (unlike risk assessment, which is personalized).

## Decision Drivers
- **Latency requirement**: Demand prediction must complete in < 10ms to fit within 120ms pricing API SLA
- **Prediction stability**: Demand patterns change gradually (15-minute granularity acceptable)
- **Computational cost**: Running complex ML models (LightGBM/LSTM) on every request is expensive
- **Context-agnostic**: Surge factors are the same for all customers at a given bay + time
- **Freshness vs. speed trade-off**: Slight staleness (up to 15 min) acceptable for demand predictions
- **Scalability**: Need to support 2K parking bays × 96 time buckets per day × multiple cities

## Options Considered

### Option 1: Real-time Inference on Every Request
Run full ML inference pipeline for each pricing request.

**Pros:**
- Always fresh predictions (no staleness)
- No pre-computation overhead
- Can incorporate truly real-time signals (e.g., live traffic)

**Cons:**
- **Unacceptable latency**: 100-200ms per request (exceeds SLA)
- **High compute cost**: ~$2K+/month for inference at scale
- **Scalability bottleneck**: Cannot handle 100K+ req/day
- **Complex real-time feature engineering**: Need streaming pipelines for events, weather
- **Wasted compute**: Same predictions recomputed many times (not customer-specific)

### Option 2: Cache Individual Predictions (Naive Memoization)
Cache predictions with key = `(parking_bay_id, time_bucket)`.

**Pros:**
- Lower latency than real-time inference (~5-10ms cache lookup)
- Simple to implement

**Cons:**
- **Cold start problem**: First request for each bay × time combination pays full latency cost
- **Incomplete coverage**: Cannot pre-populate cache for all future time buckets
- **Cache fragmentation**: 2K bays × 96 time buckets × 7 days = 1.3M cache keys (memory inefficient)
- **Inconsistent latency**: Unpredictable performance (cache hit vs. miss)

### Option 3: Pre-compute Demand Surge Grid with Periodic Refresh
Batch job pre-computes surge factors for all parking bays and time buckets for the next 24-48 hours.

**Pros:**
- **Optimal latency**: Simple Redis lookup in 2-5ms
- **Consistent performance**: No cache misses (all combinations pre-computed)
- **Cost-effective**: Batch inference amortizes cost across all predictions
- **Complete coverage**: All future time windows pre-populated
- **Predictable**: Easy to monitor and debug

**Cons:**
- **Stale predictions**: Up to 15 minutes old (acceptable for demand forecasting)
- **Batch job dependency**: System depends on periodic job success
- **Memory overhead**: Need to store 2K bays × 96 time buckets = ~192K entries

## Decision
**We will pre-compute a demand surge grid** (Option 3) with the following architecture:

### Surge Grid Design
**Data Structure** (Redis Hash):
```
Key: surge_grid:2025-10-23:14:00
Value (Hash): {
    "bay_001": "1.2",   # 1.2x surge factor
    "bay_002": "0.9",   # 0.9x (discount)
    "bay_003": "2.5",   # 2.5x (high demand)
    ...
}
TTL: 48 hours
```

**Alternative per-bay structure** (for very large scale):
```
Key: surge_grid:bay_001:2025-10-23
Value (JSON): {
    "14:00": 1.2,
    "14:15": 1.3,
    "14:30": 1.4,
    ...
}
```

### Pre-computation Process
**Batch Job** (Databricks PySpark):
1. **Schedule**: Every 15 minutes (e.g., at :00, :15, :30, :45)
2. **Horizon**: Compute surge factors for next 24-48 hours
3. **Process**:
   - Load Demand Prediction model from MLflow
   - Generate feature matrix for all (bay, time) combinations:
     - Time features: day_of_week, hour, is_holiday, is_weekend
     - Weather forecast: temperature, precipitation (from weather API)
     - Events: nearby concerts, sports, transport disruptions (from events table)
     - Historical demand: demand_same_hour_last_week, demand_trend_7d
   - Run batch inference (LightGBM on all bays × time buckets in parallel)
   - Transform predictions to surge factors:
     ```python
     surge_factor = clip(predicted_demand / baseline_demand, 0.5, 3.0)
     ```
   - Write to Redis surge grid
4. **Duration**: ~5-8 minutes for 2K bays × 96 time buckets = 192K predictions
5. **Monitoring**: Track job duration, prediction count, model version

### API Lookup Logic
```python
def get_demand_surge(parking_bay_id: str, time: datetime) -> float:
    # Round time to nearest 15-minute bucket
    time_bucket = time.replace(minute=(time.minute // 15) * 15, second=0, microsecond=0)
    time_key = time_bucket.strftime("%Y-%m-%d:%H:%M")
    
    # Lookup in surge grid
    surge_factor = redis.hget(f"surge_grid:{time_key}", parking_bay_id)
    
    if surge_factor is None:
        # Fallback: use baseline surge (1.0x) or trigger on-demand inference
        logger.warning(f"Surge factor not found for bay={parking_bay_id}, time={time_key}")
        metrics.increment("surge_grid.miss")
        return 1.0  # Neutral surge (no adjustment)
    
    metrics.increment("surge_grid.hit")
    return float(surge_factor)
```

**Lookup latency**: 2-5ms (Redis HGET operation)

### Surge Factor Calculation
Surge factors range from 0.5x (heavy discount) to 3.0x (high demand):

```python
# Predicted demand (vehicles requested per hour at this bay)
predicted_demand = model.predict(features)

# Baseline demand (historical average for this bay + time)
baseline_demand = historical_avg[parking_bay_id][time_bucket]

# Calculate surge factor
raw_surge = predicted_demand / baseline_demand
surge_factor = clip(raw_surge, min=0.5, max=3.0)

# Apply confidence adjustment (if model uncertain, reduce surge)
if model_confidence < 0.7:
    surge_factor = 0.8 * surge_factor + 0.2 * 1.0  # Pull toward 1.0x
```

**Mapping to price adjustments** (in Pricing API):
- Surge < 0.8: -20% price adjustment (incentivize parking here)
- Surge 0.8-1.2: 0% adjustment (normal pricing)
- Surge > 1.2: +20% price adjustment (disincentivize or capture willingness-to-pay)

## Consequences

### Positive
- **Ultra-low latency**: 2-5ms lookup (vs. 100-200ms real-time inference)
- **Consistent performance**: No cache misses; 100% hit rate
- **Cost-effective**: ~$200/month compute for batch job (vs. $2K+/month real-time)
- **Complete coverage**: All future time windows pre-populated
- **Scalable**: Batch job parallelized on Databricks (can handle 10K+ bays)
- **Debuggable**: Can inspect surge grid directly; easy to validate predictions

### Negative
- **Stale predictions**: Up to 15 minutes old (demand could change suddenly)
- **Batch job dependency**: If job fails, grid becomes outdated
- **Memory overhead**: 192K entries × 100 bytes = ~20MB Redis memory (acceptable)
- **Cannot incorporate real-time signals**: E.g., sudden traffic jam, breaking news events

### Neutral
- **Fixed time granularity**: 15-minute buckets (trade-off between freshness and compute cost)
- **Forecast horizon**: 24-48 hours (limited by weather forecast accuracy)
- **Event detection lag**: Events must be in database before next batch run
