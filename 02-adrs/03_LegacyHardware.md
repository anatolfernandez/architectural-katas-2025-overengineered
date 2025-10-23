# ADR-03 Legacy Hardware

## Status
Accepted

## Context
We have all our vehicles with already preinstalled hardware and provisioned with our custom software.

## Options Considered
- add custom software with AI capabilities
- reuse existing hardware and software without AI capabilities
- add remote software upgrade to enable AI on edge in the future

## Decision Drivers
- AI safety
- update based on active development
- switching third party AI services if needed

## Decision
Keep the currently running hardware and software inside the vehicles as-is. All new AI and ML capabilities are to be deployed within the backend system.

## Consequences

### Positive
- higher safety
- less development overhead
- simplified maintenance and release process

### Negative
- unable to use edge AI capabilities
- vehicles are dependent on the cloud reliability and communication to the cloud
