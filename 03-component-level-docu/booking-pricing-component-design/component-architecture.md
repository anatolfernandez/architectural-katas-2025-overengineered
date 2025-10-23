# Booking & Pricing API - Component-Level Architecture

## Table of Contents
- [System Overview](#system-overview)
- [Pricing API Architecture](#pricing-api-architecture)
- [Booking API Architecture](#booking-api-architecture)
- [Deployment Architecture](#deployment-architecture)

---

## System Overview

### Design Principles
1. **Horizontal Scalability**: Stateless services that can scale to 100+ instances
2. **Low Latency**: Sub-200ms p95 response times through aggressive caching
3. **High Availability**: 99.99% uptime through redundancy and fault tolerance
4. **Graceful Degradation**: Continue operating with reduced functionality during failures
5. **Data Consistency**: Event-driven architecture for eventual consistency

### Traffic Patterns
- **Peak Load**: ~10,000 requests/second (search + booking combined)
- **Read-Heavy**: 95% reads (search/availability), 5% writes (bookings)
- **Geographic Distribution**: Multi-region deployment for reduced latency
- **Burst Traffic**: ~5x normal load during events (concerts, weather changes)

---

## Pricing API Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           PRICING API SERVICE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────┐         ┌──────────────────┐                        │
│  │  API Gateway   │──────▶ │  Load Balancer   │                        │
│  │  (Rate Limit)  │         │   (Round Robin)  │                        │
│  └────────────────┘         └──────────────────┘                        │
│                                      │                                  │
│                    ┌─────────────────┼─────────────────┐                │
│                    ▼                 ▼                 ▼                │
│           ┌─────────────┐   ┌─────────────┐   ┌─────────────┐           │
│           │  Pricing    │   │  Pricing    │   │  Pricing    │           │
│           │  Instance 1 │   │  Instance 2 │   │  Instance N │           │
│           └─────────────┘   └─────────────┘   └─────────────┘           │
│                    │                 │                 │                │
│                    └─────────────────┼─────────────────┘                │
│                                      │                                  │
│              ┌───────────────────────┴───────────────────────┐          │
│              │                                               │          │
│              ▼                                               ▼          │
│    ┌──────────────────┐                            ┌──────────────────┐ │
│    │  Price Cache     │                            │  Circuit Breaker │ │
│    │  (Redis Cluster) │                            │  Manager         │ │
│    │  - L1: API Cache │                            └──────────────────┘ │
│    │  - L2: Computed  │                                       │         │
│    │  - L3: Features  │                                       │         │
│    └──────────────────┘                                       │         │
│              │                                                │         │
└──────────────┼────────────────────────────────────────────────┼─────────┘
                │                                                │
                ▼                                                ▼
┌────────────────────────────┐              ┌──────────────────────────┐
│   Price Orchestrator       │              │  External Dependencies   │
├────────────────────────────┤              ├──────────────────────────┤
│                            │              │                          │
│  ┌──────────────────────┐  │              │  ┌────────────────────┐  │
│  │ Price Calculator     │  │              │  │  AI Inference API  │  │
│  │ - Base Price Logic   │  │              │  │  - Risk Score      │  │
│  │ - Multiplier Engine  │  │              │  │  - Demand Predict  │  │
│  │ - Discount Rules     │  │              │  └────────────────────┘  │
│  └──────────────────────┘  │              │                          │
│                            │              │  ┌────────────────────┐  │
│  ┌──────────────────────┐  │              │  │  Pricing Database  │  │
│  │ Parallel Fetcher     │  │              │  │  (PostgreSQL)      │  │
│  │ - Risk Score API     │◀─┼──────────────┼─▶│  - Base Rates     │  │
│  │ - Demand Predict API │  │              │  │  - Vehicle Config  │  │
│  │ - Base Price DB      │  │              │  │  - Time Multiplier │  │
│  └──────────────────────┘  │              │  └────────────────────┘  │
│                            │              │                          │
│  ┌──────────────────────┐  │              └──────────────────────────┘
│  │ Price Aggregator     │  │
│  │ - Combine Results    │  │
│  │ - Apply Business Rules│ │
│  │ - Price Validation   │  │
│  └──────────────────────┘  │
│                            │
└────────────────────────────┘
```

### Core Components

#### 1. API Gateway Layer
**Responsibilities**:
- Rate limiting (100 requests/min per user, 1000/min per IP)
- Request authentication/authorization
- Request validation and sanitization
- API versioning (v1, v2)
- Request/response logging

**Technology**:
- AWS API Gateway
- Integration with JWT/OAuth2

#### 2. Pricing Service Instances
**Responsibilities**:
- Stateless request handlers
- Coordinate pricing calculation workflow
- Manage concurrent requests efficiently
- Handle timeouts and retries

**Technology**:
- Node.js/TypeScript with Express (for high throughput)
- Container-based (Docker/Kubernetes)




#### 5. Multi-Layer Caching Strategy

**Cache Hierarchy**:

```
┌─────────────────────────────────────────────────────────────┐
│ L1: API Response Cache (Redis)                              │
│ - TTL: 60 seconds                                           │
│ - Key: hash(userId, vehicleId, destinationBay, timestamp)   │
│ - Hit Rate Target: 40%                                      │
│ - Size: ~100MB per instance                                 │
└─────────────────────────────────────────────────────────────┘
                          │ (on miss)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ L2: Computed Intermediate Results (Redis)                   │
│ - Base Price Cache: 5 min TTL                               │
│ - Risk Score Cache: 24 hour TTL                             │
│ - Surge Grid Cache: 15 min TTL                              │
│ - Hit Rate Target: ~80%                                     │
└─────────────────────────────────────────────────────────────┘
                          │ (on miss)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ L3: Database + External APIs                                │
│ - PostgreSQL (Base Prices)                                  │
│ - AI Inference API (Risk, Demand)                           │
│ - Latency: ~50-150ms                                        │
└─────────────────────────────────────────────────────────────┘
```

**Redis Cluster Configuration**:
- 3 master nodes + 3 replica nodes
- Sentinel for automatic failover
- Separate connection pools per cache layer
- Cache warming on startup


## Booking API Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          BOOKING API SERVICE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────┐         ┌──────────────────┐                        │
│  │  API Gateway   │────────▶│  Load Balancer   │                       │
│  │  (Auth/Rate)   │         │                  │                        │
│  └────────────────┘         └──────────────────┘                        │
│                                      │                                  │
│                    ┌─────────────────┼─────────────────┐                │
│                    ▼                 ▼                 ▼                │
│           ┌─────────────┐   ┌─────────────┐   ┌─────────────┐           │
│           │  Booking    │   │  Booking    │   │  Booking    │           │
│           │  Instance 1 │   │  Instance 2 │   │  Instance N │           │
│           └─────────────┘   └─────────────┘   └─────────────┘           │
│                    │                 │                 │                │
│                    └─────────────────┼─────────────────┘                │
│                                      │                                  │
└──────────────────────────────────────┼──────────────────────────────────┘
                                        │
                    ┌──────────────────┴──────────────────┐
                    │                                     │
                    ▼                                     ▼
        ┌────────────────────┐              ┌────────────────────────┐
        │ Booking Orchestrator│             │  Availability Manager  │
        ├────────────────────┤              ├────────────────────────┤
        │                    │              │                        │
        │ ┌────────────────┐ │              │ ┌────────────────────┐ │
        │ │ Search Service │ │              │ │ Inventory Cache    │ │
        │ │ - Filters      │ │              │ │ (Redis)            │ │
        │ │ - Ranking      │ │              │ │ - Vehicle Status   │ │
        │ │ - Pagination   │ │              │ │ - Geospatial Index │ │
        │ └────────────────┘ │              │ └────────────────────┘ │
        │                    │              │                        │
        │ ┌────────────────┐ │              │ ┌────────────────────┐ │
        │ │ Reservation Svc│ │              │ │ Conflict Detector  │ │
        │ │ - Locking      │ │              │ │ - Time Overlap     │ │
        │ │ - Validation   │ │              │ │ - Double Booking   │ │
        │ │ - Confirmation │ │              │ └────────────────────┘ │
        │ └────────────────┘ │              │                        │
        │                    │              └────────────────────────┘
        │ ┌────────────────┐ │
        │ │ Cancellation   │ │
        │ │ Service        │ │
        │ └────────────────┘ │
        │                    │
        │ ┌────────────────┐ │
        │ │ Extension      │ │
        │ │ Service        │ │
        │ └────────────────┘ │
        └────────────────────┘
                │
                ▼
    ┌──────────────────────────┐
    │   Booking Database       │
    │   (PostgreSQL)           │
    ├──────────────────────────┤
    │                          │
    │  Tables:                 │
    │  - bookings              │
    │  - vehicles              │
    │  - parking_bays          │
    │  - booking_history       │
    │  - reservation_locks     │
    │                          │
    │  Indexes:                │
    │  - vehicle_id + time     │
    │  - user_id + status      │
    │  - bay_id + time         │
    │  - status + created_at   │
    │                          │
    └──────────────────────────┘
                │
                ▼
    ┌──────────────────────────┐
    │   Event Stream           │
    │   (Kafka/RabbitMQ)       │
    ├──────────────────────────┤
    │                          │
    │  Topics:                 │
    │  - booking.created       │
    │  - booking.cancelled     │
    │  - booking.extended      │
    │  - vehicle.reserved      │
    │  - vehicle.released      │
    │                          │
    └──────────────────────────┘
```

### Core Components

#### 1. Booking Orchestrator

**Responsibilities**:
- Coordinate multi-step booking workflow
- Integrate with Pricing API for price quotes
- Manage state transitions
- Handle rollback on failures

**Booking State Machine**:
```
SEARCHING ──▶ PRICE_QUOTED ──▶ RESERVING ──▶ CONFIRMED
                                    │              │
                                    │              ▼
                                    │         IN_PROGRESS
                                    │              │
                                    ▼              ▼
                                FAILED        COMPLETED
                                                  │
                                                  ▼
                                              CANCELLED
```


#### 3. Reservation Service with Distributed Locking

**Challenge**: Prevent double-booking in distributed system

**Solution**: Redis-based distributed locks with TTL

#### 4. Availability Manager

**Purpose**: Maintain real-time vehicle availability index

**Architecture**:
- Redis Geospatial Index for fast location-based queries
- Event-driven cache invalidation
- Background sync from database

#### 5. Extension Service

**Requirement**: Allow customers to extend booking if vehicle not reserved

### Database Scaling

#### Read Replicas Strategy

```
                ┌─────────────────┐
                │   Write Master  │
                │  (PostgreSQL)   │
                └────────┬────────┘
                          │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   Replica 1 │ │   Replica 2 │ │   Replica 3 │
│   (Region A)│ │   (Region B)│ │   (Region C)│
└─────────────┘ └─────────────┘ └─────────────┘
        │                │              │
        └────────────────┴──────────────┘
                          │
                ┌────────▼────────┐
                │  Read Traffic   │
                │ (PgBouncer Pool)│
                └─────────────────┘
```

## Deployment Architecture

### Multi-Region Deployment

```
┌─────────────────────────────────────────────────────────────┐
│                    Global Load Balancer                     │
│                  (AWS Route53 / Cloudflare)                 │
│              Latency-based routing + Health checks          │
└──────────────────┬─────────────────┬────────────────────────┘
                    │                 │
        ┌──────────▼────────┐ ┌─────▼──────────┐
        │   Region: EUW     │ │  Region: EUN   │
        │  (Frankfurt)      │ │  (Sweden)      │
        └──────────┬────────┘ └─────┬──────────┘
                    │                │
    ┌──────────────▼────────────────▼──────────────┐
    │                                              │
    │  ┌─────────────────┐   ┌─────────────────┐   │
    │  │ Pricing API     │   │ Booking API     │   │
    │  │ (K8s Cluster)   │   │ (K8s Cluster)   │   │
    │  │ - 10 pods min   │   │ - 10 pods min   │   │
    │  │ - 50 pods max   │   │ - 50 pods max   │   │
    │  └────────┬────────┘   └────────┬────────┘   │
    │           │                     │            │
    │  ┌────────▼─────────────────────▼────────┐   │
    │  │     Redis Cluster (Regional)          │   │
    │  │     - 3 masters + 3 replicas          │   │
    │  └───────────────────────────────────────┘   │
    │                                              │
    │  ┌──────────────────────────────────────┐    │
    │  │  PostgreSQL (Multi-AZ)               │    │
    │  │  - Master + 2 Replicas               │    │
    │  │  - Cross-region replication          │    │
    │  └──────────────────────────────────────┘    │
    │                                              │
    └──────────────────────────────────────────────┘
```
