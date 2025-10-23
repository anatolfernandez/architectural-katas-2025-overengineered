# ADR-08 Booking Consistency

## Status

Accepted

## Context

The booking system is a core part of our mobility platform, allowing users to reserve vehicles. We must ensure that no two users can book the same vehicle at the same time and that booking state changes (created, active, completed, canceled) remain consistent across services such as pricing, payments, and trips.

We need a reliable way to maintain data integrity, especially during concurrent booking requests or network issues. The system must handle high concurrency while maintaining strong consistency for active bookings.

## Options Considered

• **Database-level transactions (PostgreSQL with ACID guarantees)**
  - Utilize PostgreSQL's native ACID properties for consistency
  - ACID ensures: Atomicity (all-or-nothing), Consistency (valid state), Isolation (no interference), Durability (permanent changes)
  - Implement unique constraints at database level

• **Distributed locking using Redis**
  - External locking mechanism for booking coordination
  - Requires additional infrastructure complexity

• **Event-driven booking with eventual consistency (Kafka-based)**
  - Asynchronous processing with eventual consistency model
  - Higher complexity but better scalability

## Decision Drivers

• Strong consistency is required for active bookings
• Simplicity of implementation and maintenance
• Low latency during booking creation
• Alignment with existing PostgreSQL infrastructure
• Minimize infrastructure overhead

## Decision

Use PostgreSQL transactions with ACID guarantees to manage booking state and enforce unique active booking per vehicle at the database level.

For cross-service updates (e.g., pricing, payment, trip), publish booking events to Kafka for asynchronous processing.

## Consequences

### Positive

• Strong consistency and data integrity within the booking service

• Simple implementation using existing database capabilities

• Clear separation between synchronous booking logic and async updates

• Leverages proven PostgreSQL reliability and performance

• Minimal additional infrastructure requirements

### Negative

• Cross-service data may be eventually consistent (e.g., pricing or analytics updates)

• Database load may increase under high concurrency scenarios

• Potential bottleneck at database layer during peak usage

• Limited horizontal scaling options for write operations

