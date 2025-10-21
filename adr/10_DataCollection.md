
# ADR-00 Title

## Status
Proposed

## Context
Microservice architecture defines bounded contexts within single components. It is also a transactional scope that can be easily fulfilled without having to implement a distributed transaction. Our microservices will be enhanced by AI and ML models. However, in order to produce and evolve these models, we will need to have agile access to all available data sources, simplify read data access, allow analytical layer for fast model prototyping and evaluation. There are two possible options here: each microservice will come with it's own data layer, taking care of ingestion, data processing, transformations and serving. The alternative would be to build a data component responsible for managing of all the available data. We would have the following potential data sources: 
- telemetry events
- user action history
    - bookings
    - trips
    - mobile application interaction
- event information
- news for the city

## Decision
Establish an analytical platform capturing all incoming data into a data lakehouse solution. Our solution implements medallion architecture to strictly separate layers of responsibility:
- bronze layer to store all incoming data, so that no single data point is missed
- silver layer to clean the data and prepare it for consumption
- golden layer with use-case specific tables, where each schema is designated to a specific microservice

## Consequences
We build a common data management system, which allows decoupling of data management from services, but at the same time allows flexible data access, model creation and evaluation based on the specific use-case. It also means that at any point of time a new AI use-case can be added without the need to re-ingest the new microservice with all the needed data and we are able to reduce the data layer maintenance overload.
