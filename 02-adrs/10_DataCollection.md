# ADR - Data Collection

## Status
Accepted

## Context
Microservice architecture defines bounded contexts within single components. It is also a transactional scope that can be easily fulfilled without having to implement a distributed transaction. Our microservices will be enhanced by AI and ML models. However, in order to produce and evolve these models, we will need to have agile access to all available data sources, simplify read data access, allow analytical layer for fast model prototyping and evaluation. 

## Options Considered
- each microservice will come with it's own data layer, taking care of ingestion, data processing, transformations and serving
- data component responsible for managing of all the available data

## Decision Drivers
- data lakehouse / warehouse management overhead
- variety of data sources
    - telemetry events
    - user action history
        - bookings
        - trips
        - mobile application interaction
    - event information
    - news for the city
- historical data availability
- analytical UI

## Decision
Establish an analytical platform capturing all incoming data into a data lakehouse solution. Implement medallion architecture to strictly separate layers of responsibility:
- Bronze layer to store all incoming data, so that no single data point is missed
- Silver layer to clean the data and prepare it for consumption
- Gold layer with use-case specific tables, where each schema is designated to a specific microservice

## Consequences

### Positive
- common data management system, which allows decoupling of data management from services
- flexible data access, model creation and evaluation based on the specific use-case
- at any point of time a new AI use-case can be added without the need to re-ingest the new microservice with all the needed data

### Negative
- single point of failure
