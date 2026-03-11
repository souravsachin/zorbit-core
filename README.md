# zorbit-core

Platform governance repository for the Zorbit platform.

This repository contains canonical schemas, event and privilege registries, platform specifications, and service templates used across all Zorbit platform services.

## Contents

| Directory            | Description                                      |
|----------------------|--------------------------------------------------|
| schemas/events       | Canonical event envelope and domain event schemas |
| schemas/navigation   | Navigation menu schema                           |
| schemas/api          | API request/response schema conventions          |
| event-registry       | Registry of all platform events                  |
| privilege-registry   | Registry of all platform privileges              |
| platform-spec        | Platform specification documents                 |
| service-templates    | Templates for scaffolding new services           |

## Usage

This repository is consumed by:

- **zorbit-cli** — uses templates and registries to scaffold new services
- **All platform services** — conform to schemas defined here
- **SDKs** — implement patterns defined in platform-spec

## Validation

```bash
npm install
npm run validate
```

## License

Proprietary. Internal use only.
