# ADR- 07 Telemetry Streaming Context

## Status

Accepted

## Context 

Our mobility platform collects telemetry data from vehicles including location updates, battery levels, diagnostics, and usage metrics. This data streams continuously ?? from our vehicle fleet and is essential for multiple business functions.

Telemetry is not required in real-time for the app's core operation, but it is critical for:

• **AI training** (predictive maintenance, demand forecasting, anomaly detection)
• **Operational analytics** (fleet performance, usage heatmaps, battery life patterns)  
• **Business intelligence** (route optimization, pricing strategies, capacity planning)

The system must handle high-volume data streams from potentially thousands of vehicles while ensuring reliable ingestion and durable storage with no data loss.reaming


## Options Considered

• **Kafka**
  - Use Kafka for reliable message streaming and durability
  - TypeScript consumer service for data processing and validation
  - Direct write to data lake landing zone
  - Consistent with existing backend technology stack

• **Direct API ingestion with database storage**
  - REST/HTTP endpoints for direct telemetry submission
  - Traditional database storage (PostgreSQL/MongoDB)
  - Simpler architecture but limited scalability
  - Potential bottlenecks under high vehicle load


## Decision Drivers
• Reliable ingestion with message durability and fault tolerance
• Flexibility for data validation, enrichment, and transformation
• Minimize operational complexity and maintenance overhead
• Scalability to handle increasing vehicle fleet

## Decision

Use Kafka as the message broker to read telemetry messages and write them to the landing zone in the data lake.


## Consequences

### Positive

• Reliable and scalable data ingestion pipeline
• Simple and lightweight consumer service implementation
• Flexibility for future data processing enhancements
• Message durability ensures no data loss during system issues

### Negative

• Requires ongoing Kafka cluster maintenance and monitoring
• Custom consumer service needs monitoring and alerting
• Additional infrastructure complexity compared to managed solutions
• Potential scaling bottlenecks at consumer service level