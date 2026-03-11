# Zorbit Platform Architecture Specification

## MACH Principles

Zorbit follows MACH architecture:

- **Microservices** — Each service is independently deployable and owns its data store.
- **API-first** — All capabilities are exposed via REST APIs. No service bypasses the API layer.
- **Cloud-native** — Services run in containers on Kubernetes. No host-level dependencies.
- **Headless** — Backend services are decoupled from frontends. UIs consume APIs.

## Namespace Model

All resources exist within a namespace. Namespaces enforce multi-tenancy isolation.

| Type | Code | Description              | Example   |
|------|------|--------------------------|-----------|
| G    | G    | Global (platform-wide)   | G-0000    |
| O    | O    | Organization             | O-92AF    |
| D    | D    | Department               | D-51AA    |
| U    | U    | User                     | U-81F3    |

Namespace hierarchy: G > O > D > U

Every API request must include a namespace context. Services must validate that the caller has access to the specified namespace.

## Identifier Rules

All entity identifiers use short hash format:

```
<PREFIX>-<HEX_HASH>
```

Rules:
- Immutable once assigned
- Globally unique across the platform
- Non-sequential (no auto-increment)
- 4 to 8 hex characters

Standard prefixes:

| Prefix | Entity         |
|--------|----------------|
| O      | Organization   |
| D      | Department     |
| U      | User           |
| R      | Role           |
| EV     | Event          |
| SES    | Session        |
| DOC    | Document       |
| PRV    | Privilege      |
| NAV    | Navigation     |
| PII    | PII Token      |
| REQ    | Request        |
| COR    | Correlation    |
| SVC    | Service        |
| ACT    | Action         |

## REST API Grammar

All API endpoints follow this URI structure:

```
/api/v1/{namespace}/{namespace_id}/resource
```

Examples:

```
GET    /api/v1/O/O-92AF/users
POST   /api/v1/O/O-92AF/users
GET    /api/v1/O/O-92AF/users/U-81F3
PUT    /api/v1/O/O-92AF/users/U-81F3
DELETE /api/v1/O/O-92AF/users/U-81F3
```

Rules:
- Resources are plural nouns
- No verbs in URIs
- Version prefix is always `v1`
- Namespace segment is always present
- Responses use standard error and pagination envelopes (see schemas/api/)

## Event Naming Convention

All events follow the pattern:

```
domain.entity.action
```

- **domain** — the service domain (e.g. identity, authorization, audit)
- **entity** — the resource type (e.g. user, session, policy)
- **action** — past-tense verb (e.g. created, updated, deleted, expired)

Examples:

```
identity.user.created
identity.session.expired
authorization.role.assigned
audit.entry.recorded
```

All events must conform to the canonical event envelope schema (see schemas/events/event-envelope.schema.json).

## Service Communication

- **Synchronous** — REST over HTTPS with JSON payloads.
- **Asynchronous** — Kafka topics per domain. Events use the canonical envelope.
- **Database isolation** — Each service owns its database. No shared database access.

## Security

- All requests require a valid JWT issued by zorbit-identity.
- JWTs contain namespace context and privilege claims.
- Services validate JWTs using the identity service's public key.
- Namespace verification is enforced on every request.
- PII is stored in zorbit-pii-vault; operational databases hold only PII tokens.

## Authentication Flow

1. Client redirects to `accounts.platform.com`
2. Identity service authenticates the user
3. JWT is issued with namespace and privilege claims
4. Client includes JWT in Authorization header for all API calls
5. Services validate JWT and enforce namespace + privilege rules
