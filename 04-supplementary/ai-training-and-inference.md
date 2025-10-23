# AI Training and Inference Architecture

---

## Overview

This document explains the container-level architecture for the AI/ML system that powers the dynamic pricing and risk assessment capabilities for customers and their bookings.
The architecture supports two primary AI models:

1. **Customer Risk Assessment AI (CRAI)**: Predicts customer driving risk based on historical telemetry
2. **Demand Prediction AI**: Forecasts vehicle demand at parking bays to optimize pricing

The data architecture follows a **medallion data architecture** (Bronze → Silver → Gold) in Databricks and uses **Feast** for feature management with separate offline (training) and online (inference) stores.

---

## System Context

### Business Use Case
Our AI-driven vehicle pricing should take into account the following factors: 
- **Customer risk profile**: Higher-risk drivers pay more to offset insurance/damage costs
- **Demand predictions**: Pricing adjusts based on predicted demand at destination parking bays
- **Vehicle type**: Premium vehicles have higher base rates
- **Time factors**: Rush hour, weekends, weather conditions

### ML Model Integration
When a customer requests a booking price, the system:
1. Retrieves base pricing from the database
2. Calls the **Customer Risk Assessment AI** to get a risk multiplier
3. Calls the **Demand Prediction AI** to get demand-based adjustments
4. Aggregates all factors to return a personalized price per minute

**Target latency**: 120ms - 500ms end-to-end for price calculation (depending on whether inference is cached or not)

---

## Container Catalog

### 1. **ETL Pipelines**
**Container**: Databricks (Apache Spark)  
**Technology**: PySpark, Delta Live Tables

**Purpose**: Data transformation pipeline that implements the medallion architecture

**Responsibilities**:
- **Bronze → Silver**: Cleans and validates raw data
  - Removes duplicates
  - Handles missing values
  - Standardizes formats (timestamps, coordinates)
  - Validates data quality
  
- **Silver → Gold**: Creates model-specific feature tables
  - Aggregates telemetry data into time windows (7d, 30d, 90d)
  - Computes driving behavior metrics (avg speed, harsh braking count, etc.)
  - Joins weather data with trip data
  - Creates incident history summaries
  - Builds demand prediction features (historical demand per bay, event proximity)

**Execution**: Scheduled via Databricks scheduled jobs or workflows (typically daily batch processing at 2 AM)

**Output**: Writes to Gold layer tables in Data Lakehouse

---

### 2. **Data Lakehouse**
**Container**: Databricks (Delta Lake)  
**Technology**: Delta Lake, Unity Catalog

**Purpose**: Centralized data repository organized in three layers

**Data Structure**:

#### **Bronze Layer** (Raw Data)
- Vehicle telemetry streams (GPS, speed, acceleration, battery)
- Booking transactions
- Customer profiles
- Sensor data from vehicles
- Raw event feeds

#### **Silver Layer** (Cleaned Data)
- Validated telemetry with anomalies removed
- Enriched booking data with customer history
- Deduplicated events
- Standardized schemas

#### **Gold Layer** (Feature-Ready Data)
Business-level aggregates optimized for ML:

**Tables**:
- `customer_driving_behavior`: Aggregated metrics per customer (30d, 90d windows)
  - `avg_speed_violations_30d`, `harsh_braking_count_30d`, `night_driving_pct_30d`
- `customer_incident_history`: Accident and damage records
- `parking_bay_demand_history`: Historical demand patterns per bay
- `weather_features`: Historical weather by location/time
- `public_events_calendar`: Concerts, sports, transport disruptions

**Storage Format**: Delta Lake (provides ACID transactions, time travel, schema evolution)

**Access Pattern**: 
- Read by Offline Feature Store for training data generation
- Read by batch jobs for feature materialization

---

### 3. **Public Events Crawler**
**Container**: Python (Apache Scrapy or custom scraper)  
**Technology**: Python, Requests, BeautifulSoup

**Purpose**: Ingests external event data to improve demand predictions

**Data Sources**:
- Concert and sports event calendars (Ticketmaster API, venue websites)
- Public transport disruption feeds (city transit APIs)
- Weather forecasts (OpenWeatherMap API)
- Road closure announcements (city government feeds)

**Process**:
1. Daily scheduled scraping (via Databricks Jobs or cron)
2. Extracts event metadata: location, start time, expected attendance, category
3. Geocodes event locations to parking bay proximity
4. Writes raw data to Bronze layer → `public_events_raw` table

**Output**: Feeds into ETL Pipelines for processing into Gold layer

**Why it matters**: Events significantly impact vehicle demand (e.g., concert at stadium → high demand nearby for 2-3 hours post-event)

---

### 4. **Offline Feature Store**
**Container**: Feast (Feature Store framework)  
**Technology**: Feast Python SDK, Databricks connector

**Purpose**: Feature registry and training data generation with point-in-time correctness

**Key Capabilities**:

#### **Feature Registry**
Defines all features with metadata:
```python
# Example feature definition
customer_behavior_fv = FeatureView(
    name="customer_behavior_features",
    entities=[customer],
    ttl=timedelta(days=90),
    schema=[
        Field(name="avg_speed_violations_30d", dtype=Float32),
        Field(name="harsh_braking_count_30d", dtype=Int64),
        Field(name="night_driving_pct_30d", dtype=Float32),
        Field(name="accident_count_1yr", dtype=Int64),
    ],
    source=SparkSource(
        table="gold.customer_driving_behavior",
        timestamp_field="event_timestamp",
    ),
    online=True,  # Enable materialization to online store
)
```

#### **Training Data Generation**
Creates point-in-time correct datasets to prevent data leakage:
```python
# Entity dataframe: customers + prediction timestamps
entity_df = spark.sql("""
    SELECT 
        customer_id,
        booking_timestamp as event_timestamp,
        is_high_risk as label
    FROM gold.training_labels
""").toPandas()

# Get historical features as they existed at each prediction time
training_data = feast_store.get_historical_features(
    entity_df=entity_df,
    features=[
        "customer_behavior_features:avg_speed_violations_30d",
        "customer_behavior_features:harsh_braking_count_30d",
        # ... more features
    ]
).to_df()
```

**Point-in-time correctness**: Critical for time-series ML. Ensures training data only uses features that were actually available at prediction time (e.g., when predicting risk on Jan 15, only use data from before Jan 15).

**Data Source**: Reads from Gold layer tables in Data Lakehouse via Databricks SQL Warehouse connector

**Consumers**: Model Training container reads from this store

---

### 5. **Online Feature Store**
**Container**: Redis (in-memory key-value store)  
**Technology**: Redis Cluster (3+ nodes for HA)

**Purpose**: Low-latency feature serving for real-time inference (< 10ms reads)

**Data Structure**:
```
Key: feast_project:customer_behavior_features:customer_id:CUST_12345
Value (JSON): {
    "avg_speed_violations_30d": 2.5,
    "harsh_braking_count_30d": 8,
    "night_driving_pct_30d": 0.15,
    "accident_count_1yr": 0,
    "_feast_timestamp": 1729555200
}
TTL: 90 days
```

**Key Design**: 
- For **Customer Risk AI**: Key = `customer_id` 
- For **Demand Prediction AI**: Key = `parking_bay_id`

**Materialization Process**:
- **Schedule**: Nightly batch job (2 AM) via Databricks scheduled job
- **Process**: 
  1. Databricks job triggers Feast materialization
  2. Feast reads latest features from Offline Store (Gold tables)
  3. Transforms to key-value format
  4. Writes/updates Redis (incremental updates only)
  5. Old features expire based on TTL

**Label on diagram**: "Databricks scheduled jobs copy periodically from offline feature store to"

**Access Pattern**:
- Read by AI Inference API during prediction requests
- Cache hit rate target: > 95%
- Read latency: < 5ms p99

**Fallback**: If Redis unavailable (cache miss), can read from Offline Store (slower, ~100-500ms)

---

### 6. **Model Training**
**Container**: Databricks (ML Runtime)  
**Technology**: Databricks ML Runtime, PySpark ML, scikit-learn, XGBoost, PyTorch, TensorFlow

**Purpose**: Train and evaluate ML models using historical data

**Training Process**:

#### **For Customer Risk Assessment AI (CRAI)**:
1. **Data Preparation**: 
   - Request historical features from Offline Feature Store
   - Label: `is_high_risk_driver` (1 if incident in next 30 days, 0 otherwise)
   - Features: 50-100 features from customer behavior, incident history, demographics

2. **Model Training**:
   - Algorithm: XGBoost or Random Forest (typically best for tabular data)
   - Hyperparameter tuning: GridSearchCV or Optuna
   - Cross-validation: 5-fold time-series split

3. **Evaluation**:
   - Metrics: AUC-ROC, Precision, Recall, F1-score
   - Business metric: Expected claim cost reduction

4. **Logging**: All metrics, parameters, and artifacts sent to Model Registry (MLflow)

#### **For Demand Prediction AI**:
1. **Data Preparation**:
   - Historical demand per parking bay per 15-min time bucket
   - Features: day of week, hour, weather, nearby events, holiday indicator
   - Label: Demand level (regression: vehicle requests per bay) or surge category (classification: low/medium/high)

2. **Model Training**:
   - Algorithm: Gradient Boosting, LSTM (if time-series patterns), or LightGBM
   - Separate models per city or one global model with city features

3. **Evaluation**:
   - Metrics: MAE, RMSE (regression) or Accuracy, F1 (classification)
   - Business metric: Revenue impact from dynamic pricing

**Execution**: 
- **Schedule**: Weekly retraining (Sundays at 3 AM)
- **Trigger**: Manual for experiments, automated for production retraining
- **Compute**: Distributed training on Databricks clusters (GPU instances for deep learning, CPU for tree models)

**Output**: 
- Trained model artifacts (ONNX, pickle, PyTorch SavedModel)
- Training metrics and evaluation results → Model Registry

---

### 7. **Model Registry**
**Container**: MLflow (Managed by Databricks or self-hosted)  
**Technology**: MLflow Tracking + Model Registry

**Purpose**: Centralized model versioning, metadata tracking, and lifecycle management

**Key Features**:

#### **Model Versioning**
- Each trained model receives a version number (v1, v2, v3, ...)
- Models registered under logical names:
  - `customer_risk_model`
  - `demand_prediction_model`

#### **Model Metadata**
For each model version:
- **Artifacts**: Model files (ONNX, pickle), feature importance plots, confusion matrices
- **Metrics**: AUC-ROC, precision, recall, MAE, RMSE
- **Parameters**: Hyperparameters (learning_rate, max_depth, n_estimators)
- **Lineage**: 
  - Training dataset version (Gold table snapshot timestamp)
  - Feature definitions used (Feast feature view version)
  - Code version (Git commit hash)
  - Training date and duration

#### **Lifecycle Stages**
1. **None**: Newly registered, not reviewed
2. **Staging**: Under evaluation, deployed to staging environment
3. **Production**: Active model serving predictions in production
4. **Archived**: Deprecated/retired

**Promotion Workflow**:
```
1. Data Scientist trains model → Registers to MLflow (stage: None)
2. ML Engineer evaluates → Transitions to Staging
3. Integration tests pass → Promotes to Production (archives old production model)
```

#### **A/B Testing Support**
- Deploy multiple model versions simultaneously
- Tag models: `control`, `variant_a`, `variant_b`
- AI Inference API routes traffic based on experiment configuration
- Compare performance metrics in MLflow UI

**Storage**:
- **Metadata**: Database (PostgreSQL/MySQL) or Databricks managed
- **Artifacts**: DBFS (Databricks File System) or Azure Datalake

**Consumers**: 
- AI Inference API loads production models from registry
- ML Engineers review experiments and promote models

**Label on diagram**: "saves model training metrics and artifact (ONNX PICKLE) to"

---

### 8. **AI Inference API**
**Container**: Databricks (Model Serving endpoint) or FastAPI on Kubernetes  
**Technology**: Databricks Model Serving, FastAPI, or Flask

**Purpose**: Expose REST endpoints for real-time ML predictions

**API Endpoints**:

#### **1. Customer Risk Assessment**
```
POST /api/v1/predict/customer-risk
Content-Type: application/json

Request:
{
  "customer_id": "CUST_12345"
}

Response:
{
  "customer_id": "CUST_12345",
  "risk_score": 0.73,
  "risk_multiplier": 1.2,  # 0.8 - 1.5x
  "model_version": "v5",
  "features_used": {
    "avg_speed_violations_30d": 2.5,
    "harsh_braking_count_30d": 8,
    "night_driving_pct_30d": 0.15,
    "accident_count_1yr": 0
  },
  "timestamp": "2025-10-25T14:00:10Z",
  "cache_hit": true
}
```

#### **2. Demand Prediction**
```
POST /api/v1/predict/demand
Content-Type: application/json

Request:
{
  "parking_bay_id": "bay_123",
  "prediction_time": "2025-10-25T18:00:00Z"  # End of booking window
}

Response:
{
  "parking_bay_id": "bay_123",
  "prediction_time": "2025-10-25T18:00:00Z",
  "demand_level": "high",
  "surge_factor": 1.3,  # 1.0 - 3.0x
  "price_adjustment": 0.20,  # +20%
  "confidence": 0.87,
  "contributing_factors": [
    "Concert at nearby stadium (2km away, 20k attendance)",
    "Rush hour (6pm)",
    "Rainy weather forecast (reduces scooter demand)"
  ],
  "alternative_bays": [
    {"bay_id": "bay_456", "distance_km": 0.3, "surge_factor": 1.0}
  ]
}
```

**Prediction Flow**:

#### **For Customer Risk Assessment (with caching)**:
1. **Check CRAI Cache** (Redis with dedicated namespace)
   - Key: `crai_cache:customer_id:CUST_12345`
   - TTL: 24 hours (updated nightly)
   - If **cache hit** (95% of requests): Return cached risk score (~5ms)
   
2. **If cache miss** (5% of requests):
   - Fetch features from **Online Feature Store** (Redis Feast store)
   - Load **Customer Risk Model** from Model Registry (cached in memory)
   - Run inference (10-20ms for XGBoost/Random Forest)
   - Cache result in CRAI cache
   - Return risk score

**Total latency**: 
- Cache hit: ~5-10ms
- Cache miss: ~50-80ms

#### **For Demand Prediction**:
- Uses **pre-computed surge grid** (updated every 15 minutes via batch job)
- Lookup: `surge_grid[parking_bay_id][time_bucket]`
- Very fast: ~2-5ms

**Integration with Pricing Flow** (from attached document):
```
T+60ms: Pricing API calls AI Inference API (parallel):
  - Customer risk score: /predict/customer-risk
  - Demand prediction: /predict/demand

T+100ms: Pricing API aggregates:
  final_price = base_price 
              * risk_multiplier      # from CRAI
              * demand_adjustment    # from Demand AI
              * vehicle_type_factor

T+120ms: Return price to customer
```

**Key Responsibilities**:
- Fetch features from Online Feature Store (Redis)
- Load models from Model Registry (MLflow)
- Execute predictions (CPU-based inference)
- Return predictions with metadata (features used, model version)
- Log inference requests to Inference Observability

**Deployment**:
- **Option A**: Databricks Model Serving (serverless, auto-scaling)
- **Option B**: FastAPI on Kubernetes (more control, custom logic)
- **Replicas**: 3+ for high availability
- **Auto-scaling**: Based on request rate (target: 1000 req/sec capacity)

**Label on diagram**: 
- "Exposes REST endpoints for real-time inference"
- "Returns AI-driven booking price adjustment and demand prediction"
- "Fetches features from Online Feature Store"
- "Loads models from Model Registry"
- "Logs predictions to Inference Observability"

---

### 9. **Inference Observability**
**Container**: Datadog (or Prometheus + Grafana)  
**Technology**: Datadog APM, Metrics, and Logs

**Purpose**: Monitor model performance, API health, and detect issues in production

**Monitored Metrics**:

#### **API Performance**:
- **Request rate**: Requests per second (track peak load)
- **Latency**: p50, p95, p99 response times (target: p95 < 100ms)
- **Error rate**: 4xx (client errors), 5xx (server errors)
- **Cache hit rate**: CRAI cache hits vs. misses (target: > 95%)
- **Feature fetch time**: Time to retrieve features from Redis
- **Model inference time**: Time for model.predict()

#### **Model Performance**:
- **Prediction distribution**: Histogram of risk scores (detect shifts)
- **Feature distribution**: Monitor feature values for drift
- **Confidence scores**: Track model uncertainty
- **A/B test metrics**: Compare performance of model versions

#### **Business Metrics**:
- **Conversion rate**: Do personalized prices improve booking rates?
- **Revenue impact**: Total revenue from dynamic pricing
- **Customer satisfaction**: Correlation between risk scores and actual incidents

**Dashboards**:
1. **Real-time Operations**: API health, latency, error rates
2. **Model Performance**: Prediction distributions, feature drift alerts
3. **Business Impact**: Revenue trends, conversion rates by segment

**Alerting**:
- **High latency**: p95 > 150ms for 5 minutes → Page on-call engineer
- **High error rate**: > 1% 5xx errors → Auto-rollback to previous model version
- **Feature drift**: Distribution shift detected → Notify ML team (trigger retraining)
- **Cache degradation**: Hit rate < 90% → Check Redis health

**Data Ingestion**:
- AI Inference API logs every prediction (async, non-blocking):
  ```json
  {
    "timestamp": "2025-10-25T14:00:15Z",
    "request_id": "req_abc123",
    "customer_id": "CUST_12345",
    "model_name": "customer_risk_model",
    "model_version": "v5",
    "features": {"avg_speed_violations_30d": 2.5, ...},
    "prediction": {"risk_score": 0.73, "risk_multiplier": 1.2},
    "latency_ms": 65,
    "cache_hit": true
  }
  ```

**Label on diagram**: "Stores and visualises inferences logs, latencies"

---

## Data Flows

### Flow 1: Training Pipeline (Offline)

**Cadence**: Weekly (Sundays at 3 AM) or on-demand

```
1. ETL Pipelines (scheduled daily)
   ↓ [populates silver and gold tables in]
2. Data Lakehouse (Gold layer tables)
   ↓ [read feature relevant data from gold tables into]
3. Offline Feature Store (Feast)
   - Generates point-in-time correct training datasets
   ↓ [read feature relevant data from gold tables into]
4. Model Training (Databricks)
   - Trains XGBoost/Random Forest models
   - Evaluates on validation set
   ↓ [saves model training metrics and artifact (ONNX PICKLE) to]
5. Model Registry (MLflow)
   - Registers model as new version
   - ML Engineer reviews and promotes to Production
```

**Key Points**:
- ETL runs daily to keep Gold tables fresh
- Training typically weekly (unless concept drift detected)
- Point-in-time correctness prevents data leakage
- Model Registry tracks full lineage (data, features, code, metrics)

---

### Flow 2: Feature Materialization (Offline → Online)

**Cadence**: Nightly (2 AM) via Databricks scheduled job

```
1. Databricks Scheduled Job triggers
   ↓
2. Feast Materialization Service
   - Reads latest features from Offline Feature Store
   ↓ [Databricks scheduled jobs copy periodically from offline feature store to]
3. Online Feature Store (Redis)
   - Incremental update: only changed features
   - Overwrites existing keys if feature values changed
   - Expires old features based on TTL (90 days)
```

**Why nightly?**
- Customer behavior features change slowly (daily updates sufficient)
- Reduces load on Databricks and Redis
- Fresh enough for risk assessment (risk doesn't change hourly)

**Exception**: Demand prediction surge grid updates every 15 minutes (separate job)

---

### Flow 3: Real-time Inference (Online)

**Trigger**: Customer requests booking price via Mobile App

```
1. Customer → Mobile App → Booking API → Pricing API
   ↓
2. Pricing API → AI Inference API
   POST /predict/customer-risk {customer_id}
   POST /predict/demand {parking_bay_id, time}
   ↓
3. AI Inference API:
   
   a) Customer Risk:
      - Check CRAI cache (Redis, separate namespace)
      - If miss: Fetch features from Online Feature Store (Redis Feast)
                 Load model from Model Registry (cached in memory)
                 Run inference
                 Cache result
   
   b) Demand Prediction:
      - Lookup pre-computed surge_grid (updated every 15 min)
   
   ↓ [store public events to bronze layer of]
4. AI Inference API → Inference Observability (async logging)
   - Log prediction details, latency, features used
   ↓
5. AI Inference API → Pricing API
   - Return risk_multiplier (0.8 - 1.5x)
   - Return demand_adjustment (0.8 - 1.2x)
   ↓
6. Pricing API calculates final price
   final_price = base_price * risk_multiplier * demand_adjustment * vehicle_type_factor
   ↓
7. Pricing API → Booking API → Mobile App
   - Display personalized price to customer
```

**Latency Breakdown** (target < 120ms):
- Booking API → Pricing API: 10ms
- Pricing API → AI Inference API (parallel calls): 50ms
  - CRAI cache hit: 5ms
  - Demand lookup: 2ms
- Pricing API aggregation: 10ms
- Response to customer: 10ms
- **Total**: ~80ms (well under 120ms target)

---

### Flow 4: External Data Ingestion

**Cadence**: Daily (3 AM)

```
1. Databricks Scheduled Job triggers
   ↓
2. Public Events Crawler (Python script)
   - Scrapes event calendars, transport feeds, weather APIs
   ↓ [store public events to bronze layer of]
3. Data Lakehouse (Bronze layer)
   - Stores raw event data with ingestion timestamp
   ↓
4. ETL Pipelines (triggered after crawler completes)
   - Processes events → Silver (cleaned)
   - Aggregates events by proximity to parking bays → Gold
   ↓
5. Gold layer table: public_events_calendar
   - Used by Demand Prediction model for context-aware forecasting
```

**Why this matters**: Major events (concerts, sports) drastically change demand patterns. Including event data improves demand prediction accuracy by 15-20%.

---

## Key Architectural Decisions

### 1. **Why Separate Offline and Online Feature Stores?**

**Offline Store (Databricks)**:
- Handles large-scale historical data (TBs)
- Supports complex feature engineering (Spark SQL, window functions)
- Point-in-time correctness for training
- Slow for real-time queries (100-500ms latency)

**Online Store (Redis)**:
- Ultra-low latency (< 5ms)
- Handles high throughput (10k+ req/sec)
- Limited to key-value lookups (no complex queries)
- Expensive to store full history

**Solution**: Use offline for training, materialize to online for inference

---

### 2. **Why Cache CRAI Predictions Separately?**

**Problem**: Even with Redis feature store, running XGBoost inference on every request adds latency.

**Solution**: Two-tier caching
1. **Tier 1**: CRAI cache (Redis namespace `crai_cache:*`)
   - Caches final risk scores (not just features)
   - TTL: 24 hours
   - Updated nightly via batch job that pre-computes risk for all active customers
   
2. **Tier 2**: Online Feature Store (Redis Feast)
   - Fallback if CRAI cache miss
   - Provides features for on-demand inference

**Result**: 95% of requests served in < 10ms (cache hit), 5% in < 80ms (cache miss + inference)

---

### 3. **Why Pre-compute Demand Surge Grid?**

**Problem**: Demand prediction requires:
- Real-time event detection
- Weather forecast integration  
- Complex time-series models

Running this on every request would add 100-200ms latency.

**Solution**: 
- Batch job runs every 15 minutes
- Pre-computes surge factors for all parking bays for next 24 hours
- Stores in Redis: `surge_grid[bay_id][time_bucket] = surge_factor`
- API does simple lookup (2-5ms)

**Trade-off**: Predictions slightly stale (up to 15 min old) but acceptable for this use case.

---

### 4. **Why MLflow for Model Registry?**

**Requirements**:
- Version models
- Track experiments
- A/B testing support
- Databricks integration

**MLflow Benefits**:
- Native Databricks integration (same auth, shared storage)
- Open-source (avoid vendor lock-in)
- Supports multiple model formats (ONNX, pickle, PyTorch)
- Built-in experiment comparison UI
- Model lifecycle stages (None → Staging → Production)

**Alternatives considered**: Weights & Biases (more features but SaaS)

---

### 5. **Why Datadog for Observability?**

**Requirements**:
- Monitor API performance (latency, errors)
- Track model predictions (detect drift)
- Alert on anomalies
- Visualize business metrics

**Datadog Benefits**:
- Unified platform (metrics + logs + APM)
- Pre-built dashboards for common patterns
- Flexible alerting with PagerDuty integration
- ML-based anomaly detection
- Support for custom metrics (prediction distributions)

**Alternatives**: Prometheus + Grafana (more work to set up), New Relic (similar features)

---

## Model Details

### Customer Risk Assessment AI (CRAI)

**Model Type**: XGBoost Classifier (binary classification)

**Target Variable**: `is_high_risk_driver` (1 if incident in next 30 days, 0 otherwise)

**Features** (~60 features):
- Driving behavior (30d, 90d windows): avg_speed, harsh_braking_count, acceleration_avg
- Violation history: speeding_count, parking_violations, traffic_tickets
- Incident history: accident_count, at_fault_accidents, claim_amounts
- Temporal patterns: night_driving_pct, weekend_driving_pct
- Demographics: age, years_licensed, rental_frequency

**Output**: 
- Risk probability: 0.0 - 1.0
- Mapped to multiplier: 
  - Low risk (< 0.3): 0.8x - 1.0x
  - Medium risk (0.3 - 0.7): 1.0x - 1.2x
  - High risk (> 0.7): 1.2x - 1.5x

**Performance** (validation set):
- AUC-ROC: 0.87
- Precision: 0.82 (82% of predicted high-risk are truly high-risk)
- Recall: 0.79 (catches 79% of actual high-risk drivers)

**Retraining Trigger**: 
- Scheduled: Weekly
- Ad-hoc: If feature drift detected (distribution shift > 25% PSI)

---

### Demand Prediction AI

**Model Type**: LightGBM Regressor (or multi-class classifier for low/medium/high)

**Target Variable**: `vehicle_requests_per_bay` (count) or `demand_category` (class)

**Features** (~40 features):
- Temporal: day_of_week, hour, is_holiday, is_weekend
- Historical demand: demand_same_hour_last_week, demand_trend_7d
- Weather: temperature, precipitation_forecast, is_rainy
- Events: event_within_2km, event_attendance_estimate, event_category
- Location: parking_bay_capacity, nearby_poi_density, transit_accessibility

**Output**:
- Demand level: low / medium / high
- Surge factor: 1.0x (normal) to 3.0x (extreme demand)
- Price adjustment: -20% to +20%

**Performance** (validation set):
- MAE: 2.3 vehicles (mean absolute error)
- RMSE: 3.8 vehicles
- R²: 0.76

**Update Frequency**: 
- Model retraining: Weekly
- Surge grid recomputation: Every 15 minutes

---

## Scalability Considerations

### Current Scale
- **Customers**: 500K active users
- **Vehicles**: 10K fleet (cars, vans, bikes, scooters)
- **Parking bays**: 2K locations
- **Predictions per day**: ~100K CRAI + 50K demand predictions

### Bottlenecks & Solutions

**1. Redis Memory (Online Feature Store)**
- **Current**: 100GB (500K customers × 200KB features)
- **Growth**: Linear with customer base
- **Solution**: Redis Cluster (horizontal scaling), compress feature vectors

**2. AI Inference API Throughput**
- **Current**: ~100 req/sec peak
- **Growth**: Linear with bookings
- **Solution**: Auto-scaling (3-20 pods), model caching, batch predictions

**3. Feature Materialization Time**
- **Current**: 2 hours nightly (500K customers)
- **Growth**: Linear with data volume
- **Solution**: Incremental materialization (only changed features), parallel processing

**4. Model Training Time**
- **Current**: 30 minutes (10M training samples)
- **Growth**: Linear with history
- **Solution**: Use more powerful Databricks clusters, feature sampling for large datasets

---

## Security & Compliance

### Data Privacy
- **PII Protection**: Customer IDs anonymized in logs
- **GDPR Compliance**: Right to be forgotten (delete from all stores)
- **Data Retention**: Gold layer retained 2 years, Redis 90 days

### Access Control
- **Databricks**: Role-based access (Data Scientists read-only on production data)
- **MLflow**: Separate staging/production registries
- **Redis**: Password-protected, VPC-isolated
- **API**: OAuth 2.0 for inter-service communication

### Model Security
- **Model Artifacts**: Encrypted at rest (Azure Datalake/DBFS encryption)
- **API**: Rate limiting (1000 req/min per API key)
- **Audit Logs**: All model promotions tracked in MLflow

---

## Monitoring & Alerting Strategy

### Critical Alerts (Page on-call)
1. **API Down**: > 50% error rate for 2 minutes
2. **High Latency**: p95 > 200ms for 5 minutes
3. **Model Errors**: Prediction failures > 5% for 2 minutes
4. **Redis Down**: Online Feature Store unreachable

### Warning Alerts (Slack notification)
1. **Feature Drift**: PSI > 0.25 on key features
2. **Model Performance Degradation**: AUC drops > 5% on recent data
3. **Cache Hit Rate Low**: < 90% for 30 minutes
4. **Surge Grid Stale**: Not updated in > 20 minutes

### Business Metrics Monitoring
1. **Revenue Impact**: Track daily revenue from dynamic pricing
2. **Customer Satisfaction**: Monitor complaints about "unfair pricing"
3. **Model Accuracy**: Compare predicted vs. actual incidents (weekly report)

---

## Future Enhancements

### Short-term (Next 3 months)
1. **Real-time Feature Updates**: Stream telemetry to online store (currently batch)
2. **Model Explainability**: Add SHAP values to API responses ("why this price?")
3. **Multi-model Ensembles**: Combine XGBoost + Neural Net for CRAI

### Medium-term (6-12 months)
1. **Reinforcement Learning**: Optimize pricing dynamically based on booking outcomes
2. **Personalized Demand**: Predict demand at individual customer level
3. **Automated Retraining**: Trigger retraining when drift detected (no manual intervention)

### Long-term (12+ months)
1. **Federated Learning**: Train on decentralized vehicle data (privacy-preserving)
2. **Real-time Model Updates**: Deploy models without API restart (hot-swapping)
3. **Causal Inference**: Measure true causal effect of pricing on bookings (not just correlation)

---

## Summary

This architecture provides a **robust, scalable ML platform** for a vehicle sharing service with:

**Sub-100ms inference latency** via Redis caching and pre-computed predictions  
**Point-in-time correctness** in training data (prevents data leakage)  
**Automated pipelines** for ETL, feature materialization, and model retraining  
**Full observability** with Datadog monitoring and MLflow experiment tracking  
**A/B testing support** for safe model rollouts  
**External data integration** (events, weather) for context-aware predictions  

The separation of **offline (Feast + Databricks)** and **online (Redis)** feature stores optimally balances training flexibility with inference speed, while the **CRAI cache** and **pre-computed surge grid** ensure sub-second response times for dynamic pricing.
