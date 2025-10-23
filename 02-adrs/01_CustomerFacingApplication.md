# ADR-01 Customer Facing Application

## Status
Accepted

## Context
Our company needs to concentrate on the revenue bringing medium. We need to provide an easy-to-use point of access for the customers to interact with our system.

Our target user group will plan their trips on a short notice. Decisions to book or return a vehicle come on a short notice between 0 minutes and 12 hours. Therefore, it should be easy to access our system outside the house, while commuting, or waiting on the street. 

## Options Considered
1. Web application
2. Mobile SPA (single page application)
3. Mobile application
    a. 2 codebases: Swift for iOS and Kotlin for Android
    b. React Native single codebase
    c. Flutter single codebase
4. Integration with a third-party LLM provider via MCP

## Decision Drivers
1. Big cities only - good cellar network coverage
2. Short notice booking - mobile is the main priority
3. Natural language booking capabilities via voice or text

## Decision
React Native app targeting Android and iOS devices. Clean and minimalistic booking experience. Integrated agentic interface via the app.

## Consequences

### Positive
- Covers the value proposition of our business in short term booking
- Allows hands-free booking
- Simplicity of the flow doesn't justify the cost of web-application development and support
- Single code base simplifies development and provides feature parity
- React Native developer pool enables easier hiring

### Negative
- Possible issues and hacks needed in communications between the vehicle and the app
- Missing on more tech-conservative population due to lack of web-application
- Relies heavily on mobile internet availability
