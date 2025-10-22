# Overengineered Mobility App

## Team
- [Ajmal, Zohair](https://www.linkedin.com/in/zohairajmal/)
- [Demirel, Burcu](https://www.linkedin.com/in/burcu-demirel/)
- [Fernandez, Anatol](https://www.linkedin.com/in/anatol-fernandez/)
- [Syndikus, Max](https://www.linkedin.com/in/max-syndikus-58913a114/)

## Business Model
We are building a solution for a mobility company that owns a fleet of cars, vans, scooters, and bicycles. All vehicles already have premanufactured and provisioned hardware. This business struggles with vehicle availability. Therefore, the goal of this application is to improve on already existing physical assets, while at the same time dealing with the main issues: provide a trustworthy and stable software for customers to reliably find the best fitting transportation options.

The core business features, therefore, are:
- smooth user experience
- reliable infrastructure
- optimized vehicle availability

We identified functional requirements and listed them on the [Functional Requirements]() page.

## Architecture Decisions

All architecture decisions are captured with [Architecture Decision Records](adr/README.md).

### General System Design

- [ADR-01: Customer Facing Application](01_CustomerFacingApplication.md)
- [ADR-02: Third-Party Services](./adr/02_ThirdPartyServices.md)
- [ADR-03: Legacy Hardware](./adr/03_LegacyHardware.md)
- [ADR-04: Backend Services](./adr/04_BackendServices.md)
- [ADR-05: Authentication](./adr/05_Authentication.md)
- [ADR-06: Database Choice](./adr/06_DatabaseChoice.md)
- [ADR-07: Telemetry Streaming](./adr/07_TelemetryStreaming.md)
- [ADR-08: Booking Consistency](./adr/08_BookingConsistency.md)
- [ADR-09: Pricing Availability](./adr/09_PricingAvailability.md)

### AI/ML Subsystem

- [ADR-10: Data Collection](./adr/10_DataCollection.md)
- [ADR-11: Data Serving](./adr/11_DataServing.md)


## Architecture Diagram
