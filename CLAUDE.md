# Zorbit Service: zorbit-core

## Purpose

This repository stores platform governance artifacts for the Zorbit platform.

Zorbit is a MACH-compliant shared platform infrastructure used to build enterprise applications.

## Responsibilities

- Define canonical event envelope schema
- Define navigation schema
- Define API schema conventions
- Maintain event registry (all platform events)
- Maintain privilege registry (all platform privileges)
- Store platform specification documents
- Provide service templates for new service scaffolding

## Architecture Context

This service follows Zorbit platform architecture.

Key rules:

- REST API grammar: /api/v1/{namespace}/{namespace_id}/resource
- namespace-based multi-tenancy (G, O, D, U)
- short hash identifiers (PREFIX-HASH, e.g. O-92AF)
- event-driven integration (domain.entity.action)
- service isolation

## Dependencies

Allowed dependencies:

- None (this is the root governance repo)

Forbidden dependencies:

- direct database access to other services
- cross-service code imports

## Platform Dependencies

Upstream services:
- None

Downstream consumers:
- All platform services
- zorbit-cli
- zorbit-sdk-node
- zorbit-sdk-python
- zorbit-sdk-go

## Repository Structure

- /schemas/events — canonical event envelope and per-domain event schemas
- /schemas/navigation — navigation menu schema
- /schemas/api — API request/response schema conventions
- /event-registry — registry of all platform events
- /privilege-registry — registry of all platform privileges
- /platform-spec — platform specification documents
- /service-templates — templates for scaffolding new services

## Running Locally

No runtime. This is a governance artifacts repository.

## Events Published

None (governance only).

## Events Consumed

None (governance only).

## API Endpoints

None (governance only).

## Development Guidelines

Follow Zorbit architecture rules.
