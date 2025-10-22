# ADR-02 Third-Party Services

## Status
Accepted

## Context
We need a lot of auxiliary functions in out system that are not the core to our business value proposition. Our target is to maximize user experience, which also includes system reliability and vehicle availability.

## Options Considered
- In-house development of all functions
- Single third-party provider per function
- An interface to the third-party provider with a main and a back-up option

## Decision Drivers
- Operational excellence of existing providers
- Easy substitution in case of outages
- Resource concentration on core capabilities

## Decision
Build a gateway for each side function of the application. Have a main provider and an alternative substitution in case of an outage.

Current proposition:
- Payments
    - Stripe
    - parallel: Paypal
- Maps
    - Google Maps
    - backup: OpenStreetMap
- LLM
    - Gemini
    - backup: ChatGPT

## Consequences

### Positive

### Negative

