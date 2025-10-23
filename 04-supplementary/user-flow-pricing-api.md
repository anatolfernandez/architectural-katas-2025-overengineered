## Primary User Flow: Vehicle Booking with Dynamic Pricing

### Step-by-Step Flow

#### 1. **Customer looks up available vehicles** (T+0ms)
```
Customer → Mobile App → Booking API → Booking Database
```
- Customer can select vehicle type (van/car OR bike/scooter) and optionally enter a booking window (date, time, duration) and preferred location
- Query available vehicles within the city
- Retrieve vehicle metadata (type, battery level, features)

#### 2. **Customer Opens App** (T+10ms)
```
Customer → Mobile App → Booking API
```
- Customer selects concrete vehicle, and confirms booking window (date, time, duration) and pick up location for vehicle.
The customer is incentivised to also select a destination parking bay (e.g. via priority booking, exclusive promos).
- App sends request to Booking API with payload:
  ```json
  {
    "user_id": "customer_123",
    "vehicle_id": "vehicle_001",
    "booking_starts_at": "2025-10-25T14:00:00Z",
    "pickup_location": {"lat": 51.2277, "lon": 6.7735},
    "booked_duration": "4h",
    "destination_bay": "bay_123"
  }
  ```

#### 3. **Price Calculation Request** (T+50ms)
```
Booking API → Pricing API
```
In order to present the customer with a price-per-minute, the Booking API calls the Pricing API which orchestrates multiple parallel actions:

##### 3a. **Base Price Retrieval** (T+60ms)
```
Pricing API → Pricing Database
```
- Fetch base rates for vehicle type
- Apply time-of-day multipliers
- Weekend/weekday adjustments
The result is a base price-per-minute.

##### 3b. **Customer Risk Score Lookup** (T+60ms parallel)
```
Pricing API → AI Inference API → Customer Risk Assessment AI (CRAI)
```
The AI Inference API offers a web endpoint to provide a risk score for the customer. It performs the following actions:
- Checks if customer has existing risk score in CRAI inference cache (Redis) - risk scores are updated during a nightly batch job
- If found (cache hit): retrieve score immediately (~5ms)
- If not found or expired (cache miss): Run inference for CRAI model.
  ```
  Feast Online Store (serves features) → CRAI model (e.g. neural net or random forest)
  ```
  - Load customer's historical telemetry data (last 30 days)
  - Extract features such as:
    - Average speed violations: 0-100 score
    - Harsh braking events per 100km
    - Rapid acceleration frequency
    - Night driving percentage
    - Accident history
  - Run model inference (e.g. PyTorch, Tensorflow, Sklearn)
  - Returns risk multiplier: 0.8x (safe) to 1.5x (risky)
  - Cache result in CRAI cache (TTL: 24 hours)

##### 3c. **Demand prediction** (T+60ms parallel)
We have a model that predicts excess demand per parking bay, i.e. the difference between the predicted demand vs current vehicle coverage ("demand surge grid").
This step assumes that the customer has entered a destination parking bay for which the model can predict whether we should adjust the price up or down. This way
we can incentivise the customer to park at the destination bay or a different bay. In case the vehicle is a car/van with a booking window more than 2h after the booking was made, we repeat this step when the booking window starts to tell the customer which parking bays are attrive at the moment. 
```
Pricing API → AI Inference API → Demand Prediction AI
```
- Query current surge factors for destination parking bay and surrounding bays at the booking window end time
- Check pre-computed surge grid (updated every 15 minutes):
  ```python
  surge_factor = surge_grid[parking_bay_id][time_bucket]
  ```
- Surge factors range from 1.0x to 3.0x based on:
  - Day of week, hour (e.g. rush hour in certain parts of the city)
  - Weather forecast (on rainy days scooters are less attractive)
  - Detected events within 2km radius of destination parking bay
  - Confidence of the model (e.g. there is a major public event size in the area or a disruption to public transport)
  - Historical demand patterns
- Output: predicted demand level (low/medium/high)
- Adds a price adjustment of -20% to 20% based on predicted demand

#### 4. **Price Aggregation** (T+100ms)
```
Pricing API
```
Final price calculation:
```python
final_price = base_price 
            * risk_multiplier      # 0.8x - 1.5x
            * demand_adjustment    # 0.8x - 1.2x  
            * vehicle_type_factor  # Premium vehicles cost more
```

#### 5. **Response to Customer** (T+120ms)
```
Pricing API → Booking API → Mobile App
```
Returns vehicle list with personalized prices:
```json
{
  "prices": [
    {
      "vehicle_id": "vehicle_001",
      "destination_bay": "bay_123",
      "price_per_minute": 0.45
    },
    {
      "vehicle_id": "vehicle_001", 
      "destination_bay": "bay_987",
      "price_per_minute": 0.89
    }
  ]
}
```

#### 6. **Customer Books Vehicle** (T+5000ms)
```
Customer → Mobile App → Booking API
```
- Booking confirmed at displayed price and vehicle
- Price locked for duration of trip
