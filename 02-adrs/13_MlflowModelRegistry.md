# ADR - MLflow for Model Registry and Experiment Tracking

## Status
**Accepted**

## Context
Our ML platform requires a **model registry** to manage the lifecycle of multiple ML models (Customer Risk Assessment AI, Demand Prediction AI) across development, staging, and production environments. Key requirements include:

1. **Model Versioning**: Track multiple versions of each model with metadata (training date, metrics, hyperparameters)
2. **Experiment Tracking**: Log training runs with parameters, metrics, and artifacts for comparison
3. **Lifecycle Management**: Promote models through stages (Development → Staging → Production)
4. **A/B Testing**: Deploy multiple model versions simultaneously and compare performance
5. **Artifact Storage**: Store model binaries, feature importance plots, confusion matrices
6. **Databricks Integration**: Seamless integration with our existing Databricks platform
7. **Lineage Tracking**: Link models to training data, features, and code versions
8. **Auditability**: Track who promoted which model and when

The team needs to:
- Compare 10-20 training experiments per week per model
- Promote new models to production weekly (after validation)
- Roll back to previous versions if issues arise
- Run A/B tests comparing 2-3 model versions

## Decision Drivers
- **Databricks compatibility**: Native integration preferred (shared auth, storage)
- **Open-source**: Avoid vendor lock-in; must be self-hostable
- **Model format support**: ONNX, pickle, PyTorch, TensorFlow, scikit-learn
- **Ease of use**: Data scientists should log experiments with < 5 lines of code
- **UI for non-technical users**: ML engineers need visual comparison tools
- **Cost**: Prefer managed solution (Databricks MLflow) over self-hosted infrastructure
- **Team familiarity**: Team has Python experience; prefer Python-native tools
- **Scalability**: Support 100+ experiments per month, 20+ models in registry

## Options Considered

### Option 1: Weights & Biases (W&B)
Commercial ML platform (SaaS) with experiment tracking, model registry, and collaboration features.

**Pros:**
- Rich feature set (hyperparameter sweeps, reports, dashboards)
- Beautiful UI for experiment comparison
- Strong collaboration features (comments, shared reports)
- Automatic hyperparameter tuning (Sweeps)
- Advanced visualizations (interactive plots, embeddings)

**Cons:**
- **SaaS-only**: No self-hosted option (vendor lock-in)
- **Cost**: ~$50-200/user/month = $3K-12K/year for team of 5-10
- **External dependency**: Data sent to W&B servers (potential privacy concerns)
- **No native Databricks integration**: Requires custom setup
- **Overkill**: Many features (social, reports) not needed for our use case

### Option 2: Amazon SageMaker Model Registry
AWS-native model registry integrated with SageMaker.

**Pros:**
- Integrated with AWS ecosystem (S3, Lambda, SageMaker Pipelines)
- Managed service (no infrastructure to maintain)
- Built-in deployment features (SageMaker endpoints)

**Cons:**
- **AWS lock-in**: Cannot migrate to other clouds easily
- **No Databricks integration**: Need custom connectors
- **Complex setup**: SageMaker requires IAM roles, VPC config, etc.
- **Cost**: SageMaker endpoint hosting expensive (~$200+/month per model)
- **Limited experiment tracking**: Need separate tool for experiment comparison

### Option 3: MLflow (Open-Source + Databricks Managed) ✅
Open-source ML lifecycle platform with experiment tracking, model registry, and deployment tools.

**Pros:**
- **Native Databricks integration**: Managed MLflow included in Databricks
  - Shared authentication (no separate login)
  - Artifacts stored in DBFS (no external S3 setup)
  - Automatic tracking from Databricks notebooks
- **Open-source**: Apache 2.0 license (no vendor lock-in; can self-host if needed)
- **Multi-format support**: ONNX, pickle, PyTorch, TensorFlow, scikit-learn, custom
- **Built-in experiment comparison**: UI for side-by-side metric comparison
- **Lifecycle stages**: None → Staging → Production → Archived
- **REST API**: Programmatic access for CI/CD pipelines
- **Low overhead**: Simple API (`mlflow.log_param()`, `mlflow.log_metric()`)
- **No additional cost**: Included in Databricks license

**Cons:**
- **Basic UI**: Less polished than W&B (but sufficient for our needs)
- **No hyperparameter tuning**: Need separate tool (Optuna, Hyperopt)
- **Limited collaboration features**: No comments, shared reports (but we use Slack for collaboration)

### Option 4: Custom Solution (Build In-House)
Build custom model registry using database (PostgreSQL) + S3 for artifacts.

**Pros:**
- Full control over features and schema
- Tailored to exact requirements

**Cons:**
- **High development cost**: 2-3 months engineer time (~$50K)
- **Ongoing maintenance**: Need to maintain infrastructure, fix bugs
- **Feature parity**: Would take years to match MLflow features
- **Opportunity cost**: Engineers should focus on models, not infrastructure
- **Reinventing the wheel**: MLflow already solves this problem well

## Decision
**We will use MLflow** (Option 3), leveraging the **managed MLflow instance included with Databricks**.

### Architecture
- **Tracking Server**: Databricks-managed MLflow tracking server
- **Metadata Storage**: Databricks managed database (PostgreSQL)
- **Artifact Storage**: DBFS (`dbfs:/databricks/mlflow/artifacts`)
- **Access**: Via Databricks workspace UI or MLflow Python SDK

### Model Registry Schema
```
Models:
  - customer_risk_model
  - demand_prediction_model

Versions (per model):
  - Version 1 (archived)
  - Version 2 (archived)
  - Version 3 (production)
  - Version 4 (staging)
  - Version 5 (development)
```

### Experiment Tracking Workflow
```python
import mlflow
import mlflow.sklearn

# Set experiment
mlflow.set_experiment("/Shared/customer_risk_model")

with mlflow.start_run(run_name="xgboost_v5"):
    # Log parameters
    mlflow.log_param("max_depth", 6)
    mlflow.log_param("learning_rate", 0.1)
    mlflow.log_param("n_estimators", 100)
    
    # Train model
    model = xgb.XGBClassifier(**params)
    model.fit(X_train, y_train)
    
    # Log metrics
    mlflow.log_metric("train_auc", 0.89)
    mlflow.log_metric("val_auc", 0.87)
    mlflow.log_metric("precision", 0.82)
    mlflow.log_metric("recall", 0.79)
    
    # Log artifacts
    mlflow.log_artifact("feature_importance.png")
    mlflow.log_artifact("confusion_matrix.png")
    
    # Log model
    mlflow.sklearn.log_model(model, "model")
```

### Model Promotion Workflow
```python
from mlflow.tracking import MlflowClient

client = MlflowClient()

# Register model from experiment run
model_uri = f"runs:/{run_id}/model"
mv = mlflow.register_model(model_uri, "customer_risk_model")
print(f"Registered model version: {mv.version}")

# Transition to staging for validation
client.transition_model_version_stage(
    name="customer_risk_model",
    version=5,
    stage="Staging"
)

# After validation, promote to production
client.transition_model_version_stage(
    name="customer_risk_model",
    version=5,
    stage="Production",
    archive_existing_versions=True  # Archive old production model
)
```

### Model Loading in Inference API
```python
import mlflow.pyfunc

# Load production model (cached in memory)
model = mlflow.pyfunc.load_model(
    "models:/customer_risk_model/Production"
)

# Make prediction
prediction = model.predict(features)
```

### A/B Testing Support
Deploy multiple versions simultaneously with traffic splitting:

```python
# Tag models with experiment names
client.set_model_version_tag(
    name="customer_risk_model",
    version=4,
    key="ab_test",
    value="control"
)

client.set_model_version_tag(
    name="customer_risk_model",
    version=5,
    key="ab_test",
    value="variant_a"
)

# In inference API: route traffic based on customer_id hash
def get_model_version(customer_id: str) -> str:
    if hash(customer_id) % 100 < 50:  # 50% traffic
        return "models:/customer_risk_model@control"
    else:
        return "models:/customer_risk_model@variant_a"
```

## Consequences

### Positive
- ✅ **Zero setup cost**: Managed MLflow included in Databricks (no infrastructure)
- ✅ **Native integration**: Seamless with Databricks notebooks and jobs
- ✅ **Low learning curve**: Simple API; team productive in 1 week
- ✅ **Comprehensive tracking**: Logs params, metrics, artifacts, code versions automatically
- ✅ **Flexible model formats**: Supports all our model types (XGBoost, PyTorch, ONNX)
- ✅ **No vendor lock-in**: Open-source; can self-host if we leave Databricks
- ✅ **REST API**: Easy to integrate with CI/CD pipelines
- ✅ **Auditability**: Tracks all model transitions with timestamps and users

### Negative
- ❌ **Basic UI**: Less polished than W&B (but sufficient for our needs)
- ❌ **Limited collaboration**: No in-platform comments or reports (use Slack instead)
- ❌ **No built-in hyperparameter tuning**: Need Optuna or Hyperopt (minor inconvenience)
- ❌ **Databricks dependency**: If we migrate off Databricks, need to self-host MLflow

### Neutral
- ⚠️ **Model versioning strategy**: Need to define versioning conventions (e.g., major.minor for breaking changes)
- ⚠️ **Artifact cleanup**: Old experiment artifacts accumulate; need retention policy (keep last 90 days)
- ⚠️ **Access control**: Databricks workspace RBAC controls MLflow access (adequate but not fine-grained)

### Comparison: MLflow vs. W&B vs. SageMaker
| Feature | MLflow (Databricks) | Weights & Biases | SageMaker |
|---------|---------------------|------------------|-----------|
| **Cost** | ✅ $0 (included) | ❌ $3-12K/year | ⚠️ $2-5K/year |
| **Databricks Integration** | ✅ Native | ❌ Custom | ❌ Custom |
| **Open-Source** | ✅ Yes | ❌ SaaS-only | ❌ AWS-only |
| **Experiment Tracking** | ✅ Yes | ✅ Yes (richer) | ⚠️ Basic |
| **Model Registry** | ✅ Yes | ✅ Yes | ✅ Yes |
| **A/B Testing** | ✅ Yes (manual) | ✅ Yes (built-in) | ⚠️ Via SageMaker |
| **UI Quality** | ⚠️ Basic | ✅ Excellent | ⚠️ Complex |
| **Hyperparameter Tuning** | ❌ Need Optuna | ✅ Built-in Sweeps | ✅ Built-in |
| **Team Familiarity** | ✅ Python-native | ✅ Python-native | ⚠️ AWS-specific |

### Risks and Mitigations
| Risk | Impact | Mitigation |
|------|--------|-----------|
| Databricks outage | High - Cannot train/deploy models | Accept risk (Databricks SLA: 99.9%); have rollback plan for API |
| MLflow API breaking changes | Low - Rare for stable APIs | Pin MLflow version; test upgrades in staging |
| Artifact storage costs | Low - DBFS storage grows | Implement retention policy (delete artifacts > 90 days old) |
| Access control too coarse | Medium - All workspace users see all experiments | Use separate Databricks workspaces for dev/staging/prod if needed |
| Model version confusion | Medium - Accidental production deployment | Require approval for production transitions; use staging environment |

### Monitoring and Operations
**Metrics to track**:
- Number of experiments per week (target: 10-20)
- Number of model versions per model (target: < 20 active versions)
- Time to promote model (target: < 3 days from training to production)
- Artifact storage usage (target: < 500GB)

**Operational tasks**:
- Weekly: Review and archive old experiments (> 90 days)
- Monthly: Compare model performance (production vs. staging)
- Quarterly: Audit model registry (remove unused models)

### Action Items
1. ✅ Enable managed MLflow in Databricks workspace
2. ✅ Create shared experiments: `/Shared/customer_risk_model`, `/Shared/demand_prediction_model`
3. ✅ Document model logging standards (required metrics, naming conventions)
4. ✅ Train team on MLflow API (1-hour workshop)
5. ✅ Set up artifact retention policy (delete > 90 days via Databricks job)
6. 🔲 Integrate MLflow with CI/CD pipeline (auto-deploy staging models to testing environment)
7. 🔲 Create Grafana dashboard for model performance over time
8. 🔲 Implement model approval workflow (require 2 approvals for production promotion)

### Future Enhancements
1. **Auto-tagging**: Automatically tag runs with Git commit, branch, dataset version
2. **Model performance dashboard**: Track production model metrics over time (integrated with Datadog)
3. **Automated rollback**: If production model error rate > 5%, auto-rollback to previous version
4. **Multi-region registry**: Replicate model artifacts across regions for disaster recovery

## Related Decisions
- ADR-001: Separate offline and online feature stores (model lineage includes feature definitions)
- ADR-002: Two-tier caching for CRAI (model version tracked in cache)
- ADR-003: Pre-compute demand surge grid (Demand Prediction model loaded from MLflow)
- ADR-005: Datadog for observability (model performance metrics sent to Datadog)

## References
- [MLflow Documentation](https://mlflow.org/docs/latest/index.html)
- [Databricks MLflow Guide](https://docs.databricks.com/mlflow/index.html)
- [MLflow Model Registry](https://mlflow.org/docs/latest/model-registry.html)
- [Experiment Tracking Best Practices](https://mlflow.org/docs/latest/tracking.html)
