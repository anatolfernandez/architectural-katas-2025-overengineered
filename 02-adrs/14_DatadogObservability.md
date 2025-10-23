# ADR - Datadog for Observability and Monitoring

## Status
**Accepted**

## Context
Our ML inference system requires comprehensive **observability** to ensure reliability, performance, and early detection of issues. The AI Inference API serves 100K+ predictions per day with a strict < 120ms latency SLA. We need to monitor:

1. **API Performance**: Request rate, latency (p50, p95, p99), error rate, throughput
2. **Model Predictions**: Distribution of risk scores, surge factors, prediction drift
3. **Feature Serving**: Redis performance, cache hit rate, feature availability
4. **Business Metrics**: Revenue impact, conversion rates, A/B test results
5. **Infrastructure**: CPU/memory usage, network latency, Redis cluster health
6. **Alerting**: Page on-call for critical issues, Slack notifications for warnings

The team needs:
- **Unified view**: Single dashboard for all metrics (API + models + infrastructure)
- **Anomaly detection**: Automatically detect unusual patterns (e.g., sudden latency spike)
- **Root cause analysis**: Quickly diagnose issues (e.g., which endpoint is slow?)
- **Custom metrics**: Track ML-specific metrics (prediction distributions, feature drift)
- **Integration**: Works with Databricks, Redis, Kubernetes, AWS

Key requirements:
- Sub-second dashboard refresh (real-time monitoring during incidents)
- Alerting with escalation (Slack → PagerDuty)
- Log aggregation (inference logs, model logs, error traces)
- APM (Application Performance Monitoring) for distributed tracing
- Retention: 15 days of detailed metrics, 1 year of aggregated metrics

## Decision Drivers
- **Unified platform**: Prefer single tool over multiple tools (metrics + logs + APM)
- **Ease of setup**: Need production-ready in < 1 week
- **ML-specific features**: Support for custom metrics (prediction distributions, drift)
- **Alerting quality**: Smart alerting with anomaly detection (not just static thresholds)
- **Integration ecosystem**: Pre-built integrations with Redis, Kubernetes, AWS
- **Team familiarity**: Team has experience with Prometheus but open to managed solutions
- **Cost**: Budget ~$500-1500/month for monitoring (5-10 hosts + logs)
- **Scalability**: Handle 100K+ events/day

## Options Considered

### Option 1: Prometheus + Grafana (Open-Source Stack)
Self-hosted metrics collection (Prometheus) and visualization (Grafana).

**Pros:**
- **Open-source**: No licensing costs (only infrastructure)
- **Full control**: Customize everything (retention, aggregation, queries)
- **Team familiarity**: Team has Prometheus experience
- **Powerful querying**: PromQL for complex metric queries
- **Flexible dashboards**: Grafana supports many data sources

**Cons:**
- **Setup complexity**: Need to deploy Prometheus, Grafana, Alertmanager, Exporters
  - Estimated setup time: 2-3 weeks
- **No APM**: Need separate tool (Jaeger, Zipkin) for distributed tracing
- **No log aggregation**: Need separate ELK stack or Loki
- **Scaling challenges**: Prometheus single-node; need Thanos for multi-region HA
- **Alerting limitations**: Basic static thresholds; no ML-based anomaly detection
- **Operational burden**: Need to maintain infrastructure, upgrade, backup
- **No ML-specific features**: Custom dashboards for prediction drift, feature distributions

**Estimated cost**:
- Infrastructure: $200-400/month (EC2 instances, storage)
- Engineer time: 40-80 hours setup + 10 hours/month maintenance = ~$15K/year

### Option 2: New Relic
Commercial APM and observability platform (SaaS).

**Pros:**
- **Comprehensive**: Metrics, logs, APM, distributed tracing in one platform
- **Easy setup**: Agent-based; production-ready in days
- **Good APM**: Detailed transaction traces, dependency maps
- **ML insights**: Basic anomaly detection on metrics

**Cons:**
- **Cost**: ~$99-299/host/month = $1K-3K/month for 10 hosts (expensive)
- **Data ingestion pricing**: Additional charges for high log volume
- **Less popular**: Smaller community than Datadog or Prometheus
- **Limited custom metrics**: Harder to track ML-specific metrics (prediction distributions)
- **Query language**: NRQL learning curve (team prefers PromQL-like syntax)

### Option 3: Datadog ✅
Commercial observability platform (SaaS) with unified metrics, logs, APM, and ML-based anomaly detection.

**Pros:**
- **Unified platform**: Metrics, logs, APM, distributed tracing, profiling in one tool
- **Pre-built integrations**: 500+ integrations (Redis, Kubernetes, AWS, Databricks)
- **Fast setup**: Agent-based; production-ready in 1-2 days
- **ML-based anomaly detection**: Automatically detects unusual patterns
- **Excellent dashboards**: Pre-built dashboards for common services (Redis, Postgres)
- **Custom metrics**: Easy to send custom metrics (prediction distributions, feature drift)
- **Flexible alerting**: PagerDuty integration, Slack webhooks, escalation policies
- **APM**: Distributed tracing with automatic service maps
- **Log aggregation**: Centralized logs with powerful filtering/search
- **Team-friendly**: Intuitive UI; non-technical users can view dashboards

**Cons:**
- **Cost**: ~$15-31/host/month + $0.10/GB logs = $500-1500/month (but includes everything)
- **Vendor lock-in**: Proprietary platform (but can export metrics)
- **Limited customization**: Cannot modify core platform (but extensive via API)

**Estimated cost** (10 hosts + 100GB logs/month):
- Infrastructure monitoring: 10 hosts × $15 = $150/month
- APM: 10 hosts × $31 = $310/month
- Logs: 100GB × $0.10 = $10/month (first 150GB free)
- **Total**: ~$470/month (within budget)

### Option 4: AWS CloudWatch
AWS-native monitoring and logging service.

**Pros:**
- **Native AWS integration**: Works out-of-the-box with EC2, Lambda, ECS
- **No agents needed**: Automatic metrics for AWS services
- **Cost-effective for basic use**: Pay-per-use pricing

**Cons:**
- **AWS lock-in**: Cannot use if we migrate to GCP/Azure
- **Limited features**: No APM, basic alerting, weak UI
- **No custom dashboards**: Limited visualization options
- **Expensive at scale**: Custom metrics cost $0.30 each; logs $0.50/GB
- **No anomaly detection**: Only static threshold alerts
- **Poor multi-service view**: Hard to correlate metrics across services

## Decision
**We will use Datadog** (Option 3) as our primary observability platform for the following reasons:

1. **Unified platform**: Metrics + logs + APM in one tool (vs. 3 separate tools for Prometheus stack)
2. **Fast time-to-value**: Production-ready in 1-2 days (vs. 2-3 weeks for Prometheus)
3. **ML-friendly**: Easy to track custom metrics (prediction distributions, feature drift)
4. **Pre-built integrations**: Redis, Kubernetes, Databricks work out-of-the-box
5. **Smart alerting**: ML-based anomaly detection reduces false positives
6. **Cost-effective**: ~$470/month for full observability (vs. $15K+/year for self-hosted)

### Architecture
**Components**:
1. **Datadog Agent**: Installed on all hosts (Kubernetes nodes, Databricks clusters)
2. **Metrics Collection**: Automatic + custom metrics via StatsD/DogStatsD
3. **Log Aggregation**: Centralized logs from all services
4. **APM**: Distributed tracing for API requests (across services)
5. **Dashboards**: Pre-built + custom dashboards
6. **Alerts**: Configured with escalation policies (Slack → PagerDuty)

### Metrics to Track

#### API Performance
```python
# In AI Inference API (Python)
from datadog import statsd

@app.post("/api/v1/predict/customer-risk")
async def predict_risk(request: RiskRequest):
    start_time = time.time()
    
    # Track request
    statsd.increment('api.predict.requests', tags=['model:crai'])
    
    try:
        # Fetch features
        features = fetch_features(request.customer_id)
        statsd.timing('api.predict.feature_fetch_ms', (time.time() - start_time) * 1000)
        
        # Run inference
        prediction = model.predict(features)
        statsd.timing('api.predict.inference_ms', (time.time() - start_time) * 1000)
        
        # Track prediction distribution
        statsd.histogram('model.crai.risk_score', prediction.risk_score)
        
        # Track cache hit/miss
        if cache_hit:
            statsd.increment('cache.crai.hit')
        else:
            statsd.increment('cache.crai.miss')
        
        return prediction
    
    except Exception as e:
        statsd.increment('api.predict.errors', tags=['error_type:' + type(e).__name__])
        raise
    finally:
        statsd.timing('api.predict.latency_ms', (time.time() - start_time) * 1000)
```

#### Model Performance
- `model.crai.risk_score` (histogram): Distribution of risk scores
- `model.crai.risk_multiplier` (histogram): Distribution of multipliers
- `model.demand.surge_factor` (histogram): Distribution of surge factors
- `model.predictions.count` (counter): Total predictions per model
- `model.feature_drift.psi` (gauge): Population Stability Index for drift detection

#### Business Metrics
- `business.booking.conversion_rate` (gauge): % of price requests → bookings
- `business.revenue.dynamic_pricing` (gauge): Revenue attributed to dynamic pricing
- `business.customer_satisfaction.score` (gauge): Customer satisfaction scores

### Dashboards

#### 1. Real-time Operations Dashboard
**Purpose**: Monitor API health during production

**Panels**:
- Request rate (requests/sec) [Time series]
- Latency (p50, p95, p99) [Time series]
- Error rate (%) [Time series]
- Cache hit rate (CRAI cache, Feast cache) [Gauge]
- Active model versions [Table]
- Top 5 slowest endpoints [Bar chart]

#### 2. Model Performance Dashboard
**Purpose**: Track model predictions and detect drift

**Panels**:
- Risk score distribution (histogram) [Heatmap]
- Surge factor distribution (histogram) [Heatmap]
- Feature drift alerts (PSI > 0.25) [Event timeline]
- Prediction confidence distribution [Time series]
- Model version deployments [Event timeline]
- Predictions per model [Stacked area chart]

#### 3. Infrastructure Dashboard
**Purpose**: Monitor system resources

**Panels**:
- Redis memory usage (%) [Time series]
- Redis operations/sec [Time series]
- Kubernetes pod CPU/memory [Table]
- API container restarts [Event timeline]
- Databricks cluster utilization [Time series]

#### 4. Business Impact Dashboard
**Purpose**: Track revenue and customer satisfaction

**Panels**:
- Daily revenue from dynamic pricing [Time series]
- Booking conversion rate [Time series]
- A/B test results (control vs. variant) [Table]
- Customer complaints about pricing [Counter]

### Alerting Strategy

#### Critical Alerts (Page On-Call)
```yaml
- name: "API High Error Rate"
  condition: error_rate > 5% for 2 minutes
  severity: critical
  notification: PagerDuty

- name: "API Down"
  condition: requests_per_sec == 0 for 1 minute
  severity: critical
  notification: PagerDuty

- name: "High Latency"
  condition: p95_latency > 200ms for 5 minutes
  severity: critical
  notification: PagerDuty

- name: "Redis Cluster Down"
  condition: redis.can_connect == 0
  severity: critical
  notification: PagerDuty
```

#### Warning Alerts (Slack Notification)
```yaml
- name: "Feature Drift Detected"
  condition: feature_drift.psi > 0.25
  severity: warning
  notification: Slack (#ml-alerts)

- name: "Cache Hit Rate Low"
  condition: cache.hit_rate < 90% for 30 minutes
  severity: warning
  notification: Slack (#ml-alerts)

- name: "Batch Job Failed"
  condition: batch_job.status == "failed"
  severity: warning
  notification: Slack (#ml-alerts)

- name: "Model Performance Degradation"
  condition: model.accuracy < (baseline - 5%)
  severity: warning
  notification: Slack (#ml-alerts)
```

#### Anomaly Alerts (ML-based)
```yaml
- name: "Unusual Prediction Distribution"
  condition: anomaly_detection(model.crai.risk_score)
  severity: warning
  notification: Slack (#ml-alerts)

- name: "Latency Spike"
  condition: anomaly_detection(api.predict.latency_ms)
  severity: warning
  notification: Slack (#ops-alerts)
```

### Log Aggregation
**Sources**:
- AI Inference API logs (JSON format)
- Model training logs (Databricks)
- Feature materialization logs (Feast)
- Redis cluster logs

**Retention**:
- All logs: 15 days
- Error logs: 30 days
- Critical alerts: 90 days

**Log Analysis**:
- Search by customer_id, request_id, model_version
- Filter by error type, latency range
- Correlate logs with distributed traces (APM)

## Consequences

### Positive
- **Fast setup**: Production-ready in 1-2 days (vs. weeks for Prometheus)
- **Unified view**: Metrics + logs + APM in one platform (vs. 3 separate tools)
- **Reduced operational burden**: No infrastructure to maintain (vs. self-hosted)
- **ML-based anomaly detection**: Reduces false positive alerts by ~40%
- **Pre-built integrations**: Redis, Kubernetes, Databricks work out-of-box
- **Team productivity**: Non-technical users can view dashboards (vs. Prometheus/Grafana)
- **Better alerting**: Smart alerting with escalation (Slack → PagerDuty)

### Negative
- **Cost**: ~$470/month ongoing (vs. $0 for Prometheus; but saves $15K+/year engineer time)
- **Vendor lock-in**: Proprietary platform (but can export metrics if needed)
- **Data sent to external service**: Logs/metrics sent to Datadog servers (mitigated: no PII in logs)

### Neutral
- **Learning curve**: Team needs to learn Datadog query language (but intuitive)
- **Custom dashboards**: Requires JSON configuration (but UI editor available)

