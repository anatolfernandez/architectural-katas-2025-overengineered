## Background Flow 3: Smart Notifications

### Demand Monitoring (Every 15 minutes)
```
Smart Notification Engine → Pricing API → Accounts Database
```

1. **Low Demand Detection**
   ```python
   if current_demand < 0.4 * historical_average:
       trigger_notification_flow()
   ```

2. **Customer Segmentation**
   ```
   Account Database → ML Model → Target Segments
   ```
   - Identifies price-sensitive customers
   - Filters for:
     - Users who haven't booked in 7+ days
     - Users with high app engagement
     - Users near low-demand areas

3. **Price Comparison**
   ```python
   if ai_optimized_price < base_price * 0.85:
       eligible_for_notification = True
   ```

4. **Notification Dispatch**
   ```
   Smart Notification Engine → Firebase Cloud Messaging → Mobile Apps
   ```
   - Personalized message:
     ```
     "🎉 Special offer! Book now and save 25% on your next ride. 
     Vehicles available near you at reduced rates!"
     ```
   - Tracks open rates and conversion

5. **Feedback Loop**
   - Monitors notification → booking conversion
   - Updates ML model with response data
   - Adjusts targeting algorithm