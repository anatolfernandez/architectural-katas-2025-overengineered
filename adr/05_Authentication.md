# ADR-01 Customer Facing Application

## Status
Accepted

## Context
Our application deals with physical asset management and payments. We need to find a good way to authenticate users that is simple, but provides a level of security against API abuse and AI misusage.

## Options Considered
- phone number verification
- ID verification
- email-based account registration

## Decision Drivers
- reasonable complexity of the sign-up process
- account security delegated to a third-party provider

## Decision
Support oAuth-based sign-up and login with a variety oAuth providers, e.g. Google, Apple, Facebook. Require a phone number to utilize a physical device 2-factor authentication.

## Consequences

### Positive
- protection against malicious API usage

### Negative
- possibly vendor disabled accounts
- malicious actors can have virtual devices with virtual sim cards
