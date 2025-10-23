## Background Flow 1: Event Detection & Surge Pricing

### Continuous Web Scraping (Every 30 minutes)
```
Scrapy Crawler → External Websites → Event Detection AI
```

1. **Data Collection**
   - Scrapes 50+ event websites (Ticketmaster, local venues, sports sites)
   - Monitors social media APIs for trending events
   - Pulls public transport disruption feeds

2. **Event Classification**
   ```
   Raw Event Data → BERT Model → Event Importance Score
   ```
   - NLP processing to extract:
     - Event type (concert, sports, conference)
     - Expected attendance (S/M/L/XL)
     - Start/end times
     - Venue location

3. **Surge Grid Update**
   ```
   Event Detection AI → Geospatial Processing → Feature Store
   ```
   - Creates 500m x 500m grid cells across city
   - Calculates surge factors per cell:
     ```python
     surge = base_surge * event_importance * proximity_factor * time_decay
     ```
   - Updates Feature Store with new surge grid