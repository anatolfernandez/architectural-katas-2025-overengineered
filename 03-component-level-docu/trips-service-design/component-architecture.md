# Trips Service - Component-Level Architecture

## Table of Contents
- [System Overview](#system-overview)
- [Architecture Diagram](#architecture-diagram)
- [Trip End Flow](#trip-end-flow)
- [Async Validation Architecture](#async-validation-architecture)

---

## System Overview

### Purpose
The Trips Service manages the complete trip lifecycle with a focus on the critical **trip end flow**. When a user completes a trip, the service must:

1. **Immediately** end the trip from the user's perspective (fast response)
2. **Asynchronously** validate trip completion conditions:
   - Vehicle parked at correct location (telemetry check)
   - Photos show proper parking (AI validation)
   - No vehicle damage detected (AI validation)
3. **Update** vehicle availability when validation passes
4. **Trigger** fine service when validation fails

### Design Principles

1. **User Experience First**: Trip ends immediately for user, validation happens in background
2. **Event-Driven Architecture**: Async processing via message broker
3. **Eventual Consistency**: Vehicle availability updates after validation
4. **Saga Pattern**: Distributed transaction with compensating actions
5. **Fault Tolerance**: Retries, dead letter queues, manual intervention fallback

---

## Architecture Diagram

### High-Level Component View

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            MOBILE APP                                   │
│                         (End Trip Request)                              │
└────────────────────────────────┬────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         TRIPS API SERVICE                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  Trip End Controller                                           │     │
│  │  - Validate request                                            │     │
│  │  - Upload photos to S3                                         │     │
│  │  - Mark trip as "ENDED" (user perspective)                     │     │
│  │  - Return success to user                                      │     │
│  │  - Publish TripEndedEvent to broker                            │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐     │
│  │  Trip Completion Orchestrator                                  │     │
│  │  - Coordinates validation workflow                             │     │
│  │  - Manages trip completion state machine                       │     │
│  │  - Publishes events for validation steps                       │     │
│  └────────────────────────────────────────────────────────────────┘     │
│                                                                         │
└──────────────────────────────┬──────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        MESSAGE BROKER (Kafka)                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Topics:                                                                │
│  ├─ trip.ended                                                          │
│  ├─ validation.telemetry.requested                                      │
│  ├─ validation.telemetry.completed                                      │
│  ├─ validation.photo.requested                                          │
│  ├─ validation.photo.completed                                          │
│  ├─ validation.all.passed                                               │
│  ├─ validation.failed                                                   │
│  └─ vehicle.availability.updated                                        │
│                                                                         │
└──┬────────────────────┬────────────────────┬────────────────────┬───────┘
    │                    │                    │                    │
    │                    │                    │                    │
    ▼                    ▼                    ▼                    ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Telemetry   │  │    Photo     │  │   Booking    │  │     Fine     │
│  Validation  │  │  Validation  │  │  API         │  │   Service    │
│  Worker      │  │  Worker      │  │  Consumer    │  │   Consumer   │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
      │                   │                    │                   │
      │                   │                    │                   │
      ▼                   ▼                    │                   │
  ┌──────────────┐  ┌──────────────┐           │                   │
  │  Telemetry   │  │  AI Inference│           │                   │
  │  Database    │  │  API (Photo  │           │                   │
  │              │  │  Validation) │           │                   │
  └──────────────┘  └──────────────┘           │                   │
                                                │                   │
                          ┌────────────────────┴───────────────────┘
                          │
                          ▼
              ┌──────────────────────┐
              │  Validation Results  │
              │  Aggregator          │
              │  (Stateful)          │
              └──────────────────────┘
                        │
                        ▼
              ┌──────────────────────┐
              │  Decision Engine     │
              │  - All Pass → Vehicle│
              │    Available         │
              │  - Any Fail → Fine   │
              └──────────────────────┘
```

---

## Trip End Flow

### Sequence Diagram

```
User          Mobile App    Trips API    Message Broker    Telemetry Worker    Photo Worker    Booking Consumer    Fine Service
│                │             │               │                  │                 │                 │                │
│  End Trip      │             │               │                  │                 │                 │                │
│───────────────>│             │               │                  │                 │                 │                │
│                │ POST /trips/│               │                  │                 │                 │                │
│                │   {id}/end  │               │                  │                 │                 │                │
│                │────────────>│               │                  │                 │                 │                │
│                │             │ Upload photos │                  │                 │                 │                │
│                │             │   to S3       │                  │                 │                 │                │
│                │             │──────────┐    │                  │                 │                 │                │
│                │             │          │    │                  │                 │                 │                │
│                │             │<─────────┘    │                  │                 │                 │                │
│                │             │               │                  │                 │                 │                │
│                │             │ Update trip   │                  │                 │                 │                │
│                │             │ status=ENDED  │                  │                 │                 │                │
│                │             │──────────┐    │                  │                 │                 │                │
│                │             │          │    │                  │                 │                 │                │
│                │             │<─────────┘    │                  │                 │                 │                │
│                │             │               │                  │                 │                 │                │
│                │  200 OK     │               │                  │                 │                 │                │
│                │  Trip Ended │               │                  │                 │                 │                │
│<───────────────│<────────────│               │                  │                 │                 │                │
│                │             │               │                  │                 │                 │                │
│                │             │ Publish       │                  │                 │                 │                │
│                │             │ trip.ended    │                  │                 │                 │                │
│                │             │──────────────>│                  │                 │                 │                │
│                │             │               │                  │                 │                 │                │
│                │             │               │ Request Telemetry│                 │                 │                │
│                │             │               │   Validation     │                 │                 │                │
│                │             │               │─────────────────>│                 │                 │                │
│                │             │               │                  │                 │                 │                │
│                │             │               │ Request Photo    │                 │                 │                │
│                │             │               │   Validation     │                 │                 │                │
│                │             │               │──────────────────┼────────────────>│                 │                │
│                │             │               │                  │                 │                 │                │
│                │             │               │                  │ Query GPS       │                 │                │
│                │             │               │                  │ coordinates     │                 │                │
│                │             │               │                  │──────────┐      │                 │                │
│                │             │               │                  │          │      │                 │                │
│                │             │               │                  │<─────────┘      │                 │                │
│                │             │               │                  │                 │ Call AI API     │                │
│                │             │               │                  │                 │ for validation  │                │
│                │             │               │                  │                 │──────────┐      │                │
│                │             │               │                  │                 │          │      │                │
│                │             │               │                  │                 │<─────────┘      │                │
│                │             │               │                  │                 │                 │                │
│                │             │               │  Telemetry PASS  │                 │                 │                │
│                │             │               │<─────────────────│                 │                 │                │
│                │             │               │                  │                 │                 │                │
│                │             │               │  Photo PASS      │                 │                 │                │
│                │             │               │<─────────────────┼─────────────────│                 │                │
│                │             │               │                  │                 │                 │                │
│                │             │               │ All Validations  │                 │                 │                │
│                │             │               │    PASSED        │                 │                 │                │
│                │             │               │──────────────────┼─────────────────┼────────────────>│                │
│                │             │               │                  │                 │                 │                │
│                │             │               │                  │                 │   Update vehicle│                │
│                │             │               │                  │                 │   to AVAILABLE  │                │
│                │             │               │                  │                 │   ───────────┐  │                │
│                │             │               │                  │                 │              │  │                │
│                │             │               │                  │                 │   <──────────┘  │                │
│                │             │               │                  │                 │                 │                │
│                │             │               │                  │                 │   Invalidate    │                │
│                │             │               │                  │                 │   search cache  │                │
│                │             │               │                  │                 │                 │                │

ALTERNATIVE FLOW: Validation Failure

│                │             │               │                  │                 │                 │                │
│                │             │               │  Photo FAIL      │                 │                 │                │
│                │             │               │  (damage detected)                 │                 │                │
│                │             │               │<─────────────────┼─────────────────│                 │                │
│                │             │               │                  │                 │                 │                │
│                │             │               │ Validation       │                 │                 │                │
│                │             │               │    FAILED        │                 │                 │                │
│                │             │               │──────────────────┼─────────────────┼─────────────────┼───────────────>│
│                │             │               │                  │                 │                 │                │
│                │             │               │                  │                 │                 │  Create fine   │
│                │             │               │                  │                 │                 │  Notify maint. │
│                │             │               │                  │                 │                 │  ────────────┐ │
│                │             │               │                  │                 │                 │              │ │
│                │             │               │                  │                 │                 │  <───────────┘ │
│                │             │               │                  │                 │                 │                │
│                │             │               │                  │                 │   Vehicle stays │                │
│                │             │               │                  │                 │   UNAVAILABLE   │                │
```

### Core Components

#### 1. Trip End Controller

**Responsibilities**:
- Accept trip end requests from mobile app
- Validate trip state (must be IN_PROGRESS)
- Handle photo uploads to S3
- Update trip status to ENDED
- Return success to user immediately
- Publish event to trigger async validation


#### 2. Photo Uploader

**Purpose**: Handle photo uploads to S3 with proper organization and metadata


## Async Validation Architecture

### Validation State Machine

```
PENDING
  │
  ├──▶ VALIDATING_TELEMETRY ──▶ TELEMETRY_PASSED  ─┐
  │                                                 │
  │                                                 ├──▶ ALL_PASSED
  │                                                 │
  └──▶ VALIDATING_PHOTOS ──▶ PHOTOS_PASSED ────────┘
              │
              │
        ┌────▼────┐
        │ FAILED  │
        └─────────┘
              │
        ┌────▼────────────────┐
        │ Failure Reason:     │
        │ - WRONG_LOCATION    │
        │ - BAD_PARKING       │
        │ - DAMAGE_DETECTED   │
        │ - MISSING_PHOTOS    │
        └─────────────────────┘
```

### Validation Orchestrator

**Purpose**: Coordinate validation workflow and aggregate results


### Telemetry Validation Worker

**Purpose**: Verify vehicle is parked at correct location

### Photo Validation Worker

**Purpose**: Validate parking photos using AI for proper parking and damage detection