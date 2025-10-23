# ADR-06 Database Selection

## Status

Accepted

## Context

Our mobility platform manages multiple critical data domains that require reliable, transactional, and scalable data storage:

• **Accounts** – user identity, authentication, balances, payment details and account details
• **Bookings** – booking data, booking lifecycle, available vehicles
• **Trips** – vehicle reservations, trip management data, live status
• **Pricing** – dynamic tariffs, data for price calculation, discount structures
• **Telemetry** – periodic (non-real-time) vehicle data for monitoring and diagnostics
• **Analytics & AI** – training models and generating insights from historical operational data

We need a transactionally safe, scalable, and geospatially capable database for operational workloads. The database must handle complex relationships between entities while maintaining data integrity across concurrent operations.

Telemetry processing and AI workloads will be handled through a separate data lake + ETL pipeline architecture to avoid impacting operational performance.

## Options Considered

• **PostgreSQL (Relational, ACID-compliant)**
  - Proven relational database with reliable ACID transaction guarantees
  - ACID ensures: Atomicity (all-or-nothing), Consistency (valid state), Isolation (no interference), Durability (permanent changes)
  - Excellent support for complex queries and joins
  - PostGIS extension for geospatial operations
  - Strong consistency and transaction isolation

• **MongoDB (Document-based NoSQL)**
  - Flexible document schema for varying data structures
  - Good horizontal scaling capabilities
  - Built-in geospatial features
  - Eventual consistency model

• **MySQL (Relational)**
  - Widely adopted relational database
  - Good performance for read-heavy workloads
  - Less advanced features compared to PostgreSQL
  - Limited geospatial capabilities

## Decision Drivers

• Strong consistency and ACID transactions for financial and booking operations
• Relational data model fits business entity relationships naturally
• Complex query requirements for pricing and analytics
• Geospatial capabilities for location-based features
• Compatibility with existing TypeScript/Node.js backend stack
• Mature ecosystem and operational experience

## Decision

Use PostgreSQL as the primary database for accounts, pricing, bookings, and trip services.

**Implementation approach:**
• PostgreSQL provides ACID guarantees essential for financial transactions
• Relational structure naturally models business entity relationships
• PostGIS extension enables sophisticated geospatial operations
• Proven reliability and performance for operational workloads
• Strong integration with existing backend technology stack

## Consequences

### Positive

• Reliable data integrity with ACID transaction guarantees
• Strong schema control and data validation
• Mature ecosystem with extensive tooling and community support
• Excellent support for complex queries and analytical operations
• Built-in geospatial capabilities through PostGIS
• Proven scalability for operational workloads

### Negative

• Vertical scaling limitations compared to distributed NoSQL solutions
• Less flexible schema evolution compared to document databases
• Requires more careful capacity planning for high-growth scenarios
• Complex sharding if horizontal scaling becomes necessary