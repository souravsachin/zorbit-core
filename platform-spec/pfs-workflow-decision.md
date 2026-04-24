# PFS Workflow Engine — Decision Doc

- Status: APPROVED-PENDING-OWNER-SIGNOFF
- Date: 2026-04-23
- Author: Platform architect (Claude soldier)
- Scope: Choose the BPMN runtime that backs `zorbit-pfs-workflow_engine`
  and that the MUW-52 FQP workflows (and four other consumers) port onto.

---

## 1. Context

### 1.1 What we need to replace
MUW-52 FQP (Filters / Queues / Pipelines) is a bespoke Zorbit-only workflow
vocabulary. It works, but it has no visual modeler, no BPMN/DMN alignment,
no portability to external tools, and every new pipeline is code.

EPIC 16 already started the migration to BPMN 2.0 (see
`zorbit-pfs-workflow_engine/CLAUDE.md`, FQP -> BPMN 2.0 mapping section).
FQP HTTP surface is deprecated with a `Sunset: 2026-05-15` header. Phase 3
will rip FQP out. We need to lock the runtime choice now so Phase 3 and the
consumer ports below don't stall.

### 1.2 Consumer list (the five modules that must port onto the chosen engine)

| Consumer                              | Repo                                           | Current state       |
|---------------------------------------|------------------------------------------------|---------------------|
| hi_uw_workflow (underwriting)         | zorbit-app-uw_workflow                         | FQP, porting        |
| hi_claim_adjudication                 | zorbit-app-hi_claim_adjudication_workflow      | FQP, porting        |
| hi_claim_initiation                   | (subset of claim_adjudication)                 | FQP, porting        |
| network_empanelment                   | zorbit-app-network_empanelment_workflow        | FQP, porting        |
| pricing (UW pricing rules)            | part of hi_uw_workflow and pcg4                | hardcoded if/else   |

All five are queue-based, human-in-the-loop or mixed human+AI, and emit
audit events. None needs millisecond throughput; all need durable state,
visual editing by SMEs, and versioning.

### 1.3 Owner directive

> "I thought you had already implemented one. If that node-js based
> lightweight implementation suffices, we'll go with it. Otherwise Camunda,
> so be it. Just make sure that it integrates within our unified console."

Owner recollection verified: a Node.js `bpmn-engine` implementation already
exists and is deployed — see §2 "Forensic Evidence" below.

---

## 2. Forensic Evidence — What Already Exists

### 2.1 `zorbit-pfs-workflow_engine` repo (on laptop + deployed per MEMORY.md)

- Runtime dep: `bpmn-engine@^25.0.1` (paed0/bpmn-engine, MIT, 13-year old
  project, actively maintained).
- NestJS 10.x on Node 20.x, MongoDB storage (collections
  `bpmn_processes`, `bpmn_instances`).
- ~1149 LOC of production BPMN code split across:

| File                                  | LOC | Purpose                                   |
|---------------------------------------|-----|-------------------------------------------|
| services/bpmn-executor.service.ts     | 169 | Thin wrapper around Engine()              |
| services/bpmn-instance.service.ts     | 264 | Instance lifecycle, signal replay         |
| services/bpmn-process.service.ts      | 121 | Definition CRUD + versioning              |
| services/bpmn-seed.service.ts         | 268 | Demo + seeded BPMN loading                |
| controllers/bpmn-process.controller   |  86 | REST API                                  |
| controllers/bpmn-instance.controller  |  75 | REST API                                  |
| controllers/bpmn-seed.controller      |  34 | Seed endpoints                            |
| schemas/bpmn-process.schema.ts        |  52 | Mongoose model                            |
| schemas/bpmn-instance.schema.ts       |  80 | Mongoose model                            |

### 2.2 Seeded BPMN XML process catalog (in `seeds/bpmn/`)

`BP-CLM.bpmn`, `BP-EMP.bpmn`, `BP-MKC.bpmn`, `BP-UWC.bpmn`, `BP-UWQ.bpmn`,
plus `validate.js` and `CANDIDATE-GROUPS.md`. These cover claims, network
empanelment, make/check, UW case, and UW queue — a strong slice of the
consumer list already.

### 2.3 REST API surface (already live)

```
POST   /api/v1/O/:org/workflow/bpmn-processes           register XML
GET    /api/v1/O/:org/workflow/bpmn-processes
GET    /api/v1/O/:org/workflow/bpmn-processes/:id
PUT    /api/v1/O/:org/workflow/bpmn-processes/:id       new version
DELETE /api/v1/O/:org/workflow/bpmn-processes/:id       retire
POST   /api/v1/O/:org/workflow/bpmn-processes/:id/start
GET    /api/v1/O/:org/workflow/bpmn-instances
GET    /api/v1/O/:org/workflow/bpmn-instances/:id
GET    /api/v1/O/:org/workflow/bpmn-instances/:id/history
POST   /api/v1/O/:org/workflow/bpmn-instances/:id/signal
POST   /api/v1/O/:org/workflow/bpmn-instances/:id/cancel
```

### 2.4 Kafka events (already wired)

```
platform.workflow.process.registered
platform.workflow.process.updated
platform.workflow.process.retired
platform.workflow.instance.started
platform.workflow.instance.advanced
platform.workflow.instance.completed
platform.workflow.instance.failed
platform.workflow.instance.cancelled
```

### 2.5 Unified-console integration (already shipped)

Pages found in `zorbit-unified-console/src/pages/workflow-engine/`:
- `WorkflowEngineHubPage.tsx`
- `WorkflowSetupPage.tsx`
- `WorkflowDeploymentsPage.tsx`
- `WorkflowFiltersPage.tsx` (legacy FQP)
- `WorkflowQueuesPage.tsx` (legacy FQP)
- `WorkflowPipelinesPage.tsx` (legacy FQP)

Plus the BPMN visual designer at
`zorbit-unified-console/src/components/shared/WorkflowDesigner/`:
- `DesignerCanvas.tsx`, `DesignerToolbar.tsx`, `PropertiesPanel.tsx`,
  `bpmn-js-shims.d.ts` — uses `bpmn-js` (Camunda's MIT-licensed modeler
  component) embedded directly. No Camunda server needed.

---

## 3. Options Evaluated

### Option A — Keep the existing Node.js `bpmn-engine` in `zorbit-pfs-workflow_engine`

- **Runtime**: `bpmn-engine` (npm, MIT, paed0).
- **Modeler**: `bpmn-js` embedded in unified-console (already shipped).
- **Storage**: MongoDB (`zs-pg` equivalent for Mongo: the shared Mongo
  instance used by datatable/form_builder). Definitions + instances both
  stored, definitions versioned.
- **Deploy**: single NestJS container in the `zu-pfs` bundle. No extra
  infra. ARM64 + x86_64 Docker images (standard Node base image).

### Option B — Camunda 8 / Zeebe as shared TPM (`zs-zeebe`)

- **Runtime**: Zeebe (Go binary), official Camunda 8 SaaS / self-managed.
- **Modeler**: Camunda Desktop Modeler (SME install) or Web Modeler (paid).
- **Zorbit side**: `zorbit-pfs-workflow_engine` becomes a thin facade that
  proxies to Zeebe via `zeebe-node` gRPC client.
- **Storage**: Zeebe has its own Elasticsearch-backed state store
  (Operate + Tasklist are separate services). We'd add Zeebe-broker,
  Elasticsearch, Operate, Tasklist containers — 4 extra services minimum.
- **Deploy**: Zeebe broker runs on Linux ARM64 since Camunda 8.3+. Helm
  chart is the supported path; Docker Compose works for dev.

### Comparison

| Dimension                      | A: bpmn-engine (Node)       | B: Camunda 8 / Zeebe         |
|--------------------------------|------------------------------|-------------------------------|
| BPMN 2.0 coverage              | ~80% (seq flow, gateways, user tasks, service tasks, timers, sub-process, message) — fully sufficient for our five consumers | ~99% including DMN, compensation, multi-instance markers, call activity |
| DMN (decision tables)          | None — needs separate engine | First-class, same platform    |
| Maintenance burden             | One dep bump per year        | 5 containers (broker, ES, Operate, Tasklist, gateway); quarterly major upgrades |
| Integration cost right now     | ZERO — already built         | 6-10 engineer-weeks           |
| Visual modeler                 | `bpmn-js` embedded — already in unified-console | External Camunda Modeler app (desktop) or SaaS |
| Versioning                     | Hand-rolled in service (works) | Built-in via deploy command   |
| HA / horizontal scale          | Stateless Nest nodes + Mongo (good for our throughput — sub-100 RPS) | Built-in partition replication |
| Team familiarity               | TypeScript-first, matches rest of platform | Java/Go world, gRPC, ES — outside team's comfort zone |
| ARM64 + x86_64                 | Node base image, both archs  | Supported 8.3+, but ES is heavy on ARM |
| Unified-console fit            | Same Nest REST, embeds `bpmn-js`, matches every other PFS | Needs Operate/Tasklist iframe OR we rebuild UIs over Zeebe gRPC |
| Licensing                      | MIT                          | Source-available + commercial for Web Modeler |
| Rip-out cost if wrong          | Low — 1149 LOC, swap executor service | Catastrophic — 4 new infra pieces |

---

## 4. Verdict — Option A: keep `bpmn-engine` (Node.js)

**Decision: ship `zorbit-pfs-workflow_engine` as-is with `bpmn-engine` as
the runtime and `bpmn-js` as the visual modeler inside the unified console.
Do NOT introduce Camunda 8 / Zeebe.**

Rationale, in plain terms:

1. **It already exists and is deployed.** The owner's recollection is
   correct. 1149 LOC of BPMN code, 5 seeded processes, 8 canonical Kafka
   events, full REST API, unified-console pages — all shipped. Choosing
   Camunda means throwing that away.
2. **Our five consumers do not need what Zeebe provides.** None of them
   hits 100 RPS. None needs compensation, multi-instance parallelism, or
   DMN at scale. All are queue-based human-in-the-loop flows.
3. **Unified-console integration is already solved** via `bpmn-js`. No
   Operate/Tasklist iframes. No separate modeler app for SMEs to install.
4. **One runtime, one stack.** bpmn-engine is TypeScript/Node like the
   rest of the platform. Camunda would be the first Java/Go service on
   ARM UAT and would drag in Elasticsearch.
5. **Cheap escape hatch.** If we later hit a BPMN feature bpmn-engine
   can't do (e.g. compensation at scale), we swap out `bpmn-executor.
   service.ts` for a zeebe-node facade — the REST API and storage model
   stay identical. See §7.

---

## 5. Integration With Unified Console

Keep the existing pages under `/m/workflow_engine/*` (note the `_`
per nomenclature rule `feedback_nomenclature_strict.md`). Final set:

| Console page                          | Route                                           | Status        |
|---------------------------------------|-------------------------------------------------|---------------|
| Hub (overview cards + metrics)        | `/m/workflow_engine`                            | Exists, polish|
| Process Definitions list              | `/m/workflow_engine/processes`                  | Exists        |
| Process Definition detail + versions  | `/m/workflow_engine/processes/:bpHashId`        | Extend        |
| Visual BPMN Designer                  | `/m/workflow_engine/designer`                   | Exists (`bpmn-js`) — add SME-friendly palette |
| BPMN Preview (read-only viewer)       | `/m/workflow_engine/processes/:bpHashId/preview`| NEW — same `bpmn-js` instance in view-only mode |
| Instance Monitor list                 | `/m/workflow_engine/instances`                  | NEW           |
| Instance detail + live trace          | `/m/workflow_engine/instances/:biHashId`        | NEW — overlays on BPMN diagram, shows token position |
| Seed / demo loader                    | `/m/workflow_engine/seed`                       | Exists        |

All pages already auth via the unified-console JWT. APIs are called over
`/api/v1/O/:org/workflow/*` exactly like every other PFS. No new auth
plumbing, no iframe bridge.

---

## 6. Impact on `zorbit-pfs-rules_engine`

Owner's earlier directive: rules engine uses `json-rules-engine` (npm),
DMN as fallback if we go Camunda. We did NOT go Camunda, so:

**Keep `zorbit-pfs-rules_engine` as a separate service using
`json-rules-engine`. Do NOT consolidate into BPMN.**

Reasons:
- BPMN is for **flow**. Rules engine is for **decision logic** (pricing
  tables, eligibility checks, commission rules). Different audiences:
  workflow SMEs design BPMN, underwriting actuaries / product managers
  author rules.
- `json-rules-engine` is JSON-native, editable in a table UI without
  learning DMN FEEL expressions. Matches the datatable/form_builder
  authoring philosophy.
- BPMN service tasks in our flows can still **call** the rules engine
  via REST. Clean separation, clean audit.
- Optional: a BPMN "Business Rule Task" can resolve to a rules_engine
  ruleset hash id — cheap bridge, no coupling.

Consolidation is explicitly rejected. Two services, one call pattern.

---

## 7. Execution Plan

### 7.1 Already done (keep)

- [x] `zorbit-pfs-workflow_engine` Nest service with bpmn-engine.
- [x] `bpmn_processes` + `bpmn_instances` Mongo collections, hashId'd.
- [x] 8 canonical Kafka events.
- [x] 5 seeded BPMN XMLs (claim, empanelment, MKC, UW case, UW queue).
- [x] Unified-console Designer (`bpmn-js`).
- [x] FQP Sunset headers (2026-05-15).

### 7.2 To build — gated, in order

| # | Task                                                            | Gate to pass                                                  | Depends on |
|---|-----------------------------------------------------------------|---------------------------------------------------------------|------------|
| 1 | Instance Monitor console page (list + filter)                   | Shows all running instances for an org                        | —          |
| 2 | Instance detail page with live BPMN token overlay               | Renders token on correct activity from history[]              | 1          |
| 3 | BPMN Preview (read-only viewer) on process detail               | SMEs can inspect without edit permissions                     | —          |
| 4 | Port uw_workflow from FQP to BPMN (PoC on one queue first)      | One UW queue runs entirely off bpmn-engine                    | —          |
| 5 | Port hi_claim_adjudication from FQP to BPMN                     | Adjudication ICR loop runs off bpmn-engine                    | 4          |
| 6 | Port network_empanelment from FQP to BPMN                       | Empanelment approval chain runs off bpmn-engine               | 4          |
| 7 | Port pricing rules to `zorbit-pfs-rules_engine`                 | Rules engine deployed; pricing ruleset under admin edit       | —          |
| 8 | Wire BPMN "Business Rule Task" -> rules_engine call             | Example BPMN invokes a rules ruleset by hashId                | 7          |
| 9 | Phase 3 FQP rip-out (after 2026-05-15 sunset)                   | No FQP traffic for 14 days; delete controllers/services       | 4,5,6      |

### 7.3 Dependencies

- **Mongo** on zs-pg host (already up for datatable/form_builder).
- **Kafka** on zs-kafka (already up).
- **Identity** JWT (already integrated).
- **Unified-console** `bpmn-js` dep (already in package.json).

No new infra. No new container on zs-*.

---

## 8. Rollback / Escape Hatch

If during porting we hit a BPMN feature `bpmn-engine` truly can't do
(real-world likely candidates: compensation events at scale, complex
multi-instance parallel with aggregation, 1000+ RPS throughput):

1. **Do NOT rip out `zorbit-pfs-workflow_engine`.** Keep its API surface,
   its Mongo collections, its Kafka events — every consumer already
   depends on them.
2. **Swap only `bpmn-executor.service.ts`** (169 LOC, one file) for a
   new implementation that delegates to Zeebe via `zeebe-node` gRPC.
3. **Spin up `zs-zeebe` as shared TPM** — Zeebe broker + ES + Operate +
   Tasklist via Helm on ARM UAT.
4. **Re-deploy the same BPMN XMLs to Zeebe** via the zeebe-node `deploy`
   call. Since bpmn-engine and Zeebe both consume standard BPMN 2.0 XML
   authored by `bpmn-js`, the XMLs are portable.
5. **Consumer services see zero API change.** All five continue to use
   `/api/v1/O/:org/workflow/*`.

This escape hatch is what makes Option A safe: we've decoupled the
platform contract (REST + Kafka) from the runtime (bpmn-engine), so the
runtime is replaceable. This is exactly why we don't go Camunda now —
we don't pay the infra tax until we know we need it.

---

## 9. Approvals

- [ ] Platform owner sign-off
- [ ] Architect sign-off
- [ ] EPIC 16 Phase 3 kickoff (post sign-off)
