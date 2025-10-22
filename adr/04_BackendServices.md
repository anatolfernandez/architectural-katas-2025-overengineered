# ADR-04 Backend Services

## Status
Accepted

## Context
Our backend system will handle user flows and enable the use of AI. We want to define service borders, so that it enables effective scaling, communication, segregation of responsibility, and future extension.

## Options Considered
- monolith architecture
- microservice architecture
    - sync
    - async

## Decision Drivers
- responsive user experience
- reliable booking
- asynchronous analytics

## Decision
Microservice architecture with sync REST calls and async message publishing for analytics.

## Consequences

### Positive
- system elasticity, by scaling individual services based on current demand
- flexibility of new feature development, since the service are independently communicating through defined interfaces

### Negative
- maintenance overhead
- networking performance overhead
- possible need for distributed transactions
