## Background Flow 2: Risk Assessment Updates

### Batch Retraining (Daily at 2 AM)
```
Pricing Database → Customer Risk Assessment AI
```

1. **Data Preparation**
   - Extract all trips from last 90 days
   - Join with accident reports, violations
   - Create training dataset with 100+ features

2. **Model Training**
   ```python
   model = TensorFlow.Sequential([...])
   model.fit(X_train, y_train, validation_split=0.2)
   ```
   - Trains on millions of historical trips
   - Validates against known insurance claims

3. **Model Deployment**
   ```
    Model Registry → Production Endpoint
   ```
   - A/B testing with 5% traffic initially
   - Gradual rollout if metrics improve

### Real-time Scoring (Per trip completion)
```
Stream Processing → Risk Assessment AI → Feature Store → Account DB
```
- Processes completed trip telemetry
- Updates customer risk score
- Triggers re-pricing for future bookings