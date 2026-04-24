---
title: PMJAY claim lifecycle — module composition design
slug: pmjay-claim-module-design
version: 0.1-draft
status: draft-for-review
type: DESIGN
audience: [platform-engineers, pmjay-implementation-team]
last-verified: 2026-04-24
---

# PMJAY claim lifecycle — module composition design

PMJAY = Pradhan Mantri Jan Arogya Yojana (India's flagship government health insurance).

## Owner's guiding principle (captured 2026-04-24)

> "We need to reimagine, rebuild these modules using formbuilder, datatable, PII,
> and workflow engine (BPMN, replacement of FQP) thing. And then build custom
> pieces as required. Also, all decisioning engines are essentially rule engines
> which are fed the rules using AI or even manually and then run deterministically
> repeatedly for each case."

Translation: **business modules are thin orchestrators that compose PFS primitives.**
Custom code only where PMJAY-specific domain logic cannot be expressed declaratively.

## Business line / capability / edition

- **Edition:** Health Insurance
- **Business Line:** Servicing
- **Capability Area:** Claims Management
- **Tenant type:** Government program (PMJAY has state empanelment + central NHA oversight)

## Claim lifecycle (BPMN processes to author)

```
Pre-auth     →     Intake (FNOL)    →     Verification    →     Adjudication    →    Payment   →     Recon
(optional)         pre-treatment            evidence                 decisioning          payout          settlement
                                            KYC + docs              rules                 vendor          reports
```

Each arrow above is a BPMN sequence flow; each box is a BPMN subprocess with UserTasks.

## Module map (using existing PFS primitives)

| Stage | Module | Type | Status | Composition |
|---|---|---|---|---|
| Pre-auth | `zorbit-app-hi_preauth` (new, scaffold) | app | ❌ to scaffold | form_builder (pre-auth form) + rules_engine (pre-auth decision rules) + workflow_engine (pre-auth BPMN) + pii_vault |
| Intake (FNOL) | `zorbit-app-hi_claim_initiation` | app | ✅ exists (bare) | form_builder (FNOL form) + pixel (doc upload + OCR) + pii_vault (claimant details tokenised) + workflow_engine (BPMN: Intake → Verify → Route) + rules_engine (auto-triage + completeness checks) |
| Verification | `zorbit-pfs-kyc` | pfs | ✅ exists | OCR'd docs from pixel + KYC rules + member/provider matching |
| Adjudication | `zorbit-app-hi_claim_adjudication_workflow` | app | ✅ exists (bare) | rules_engine (adjudication rulesets per plan/package) + medical_coding (ICD/PCS codes, package code mapping) + workflow_engine (BPMN: Auto-Adjudicate → Manual Review → Decision) + datatable (adjudicator work queue) |
| Decisioning | `zorbit-app-hi_claim_decisioning` | app | ✅ exists (bare) | rules_engine (decisioning rules) + workflow signals to adjudication |
| Payment | `zorbit-pfs-payment_gateway` + `zorbit-app-hi_claim_payout` (new if needed) | pfs + app | ⚠️ payment_gateway may exist, payout to scaffold | payment_gateway (wires to NHA/govt disbursement) + audit (every transfer) |
| Reconciliation | `zorbit-app-hi_claim_payment_recon` | app | ✅ exists (bare) | datatable (recon queue) + rules_engine (recon rules) + workflow_engine (BPMN: Fetch → Match → Dispute → Close) + analytics_reporting (recon dashboards) |

## Cross-cutting PFS dependencies (used by multiple stages)

| PFS | Role | Status |
|---|---|---|
| `zorbit-pfs-form_builder` | All user-facing intake/decisioning forms (FNOL, adjudicator work-item, recon dispute) | ✅ v0.2.0 |
| `zorbit-pfs-datatable` | All list views with PII-masked columns + filter + bulk actions | ✅ v0.1.0 |
| `zorbit-pfs-pixel` | Document digitisation (scan → OCR → structured extraction) | 🟡 scaffold exists, **needs port from owner's live project** |
| `zorbit-pii-vault` | Every PII field tokenised (claimant Aadhaar, bank account, phone, address) | ✅ deployed on prior envs |
| `zorbit-pfs-rules_engine` | Every decision point (completeness, adjudication, recon) | 🟡 scaffold exists, **needs json-rules-engine impl** |
| `zorbit-pfs-workflow_engine` | Every multi-step flow (pre-auth → payment) | ✅ v1.3.0 (bpmn-engine; visual designer) |
| `zorbit-pfs-medical_coding` | ICD / PCS / PMJAY package code lookup + validation | ✅ v0.1.0 |
| `zorbit-pfs-kyc` | Member + provider KYC | ✅ exists |
| `zorbit-pfs-notification` | SMS/email to claimant, hospital, adjudicator | ✅ exists |
| `zorbit-cor-audit` | Every action logged with who/when/why/what | ✅ deployed on prior envs |
| `zorbit-cor-event_bus` | Every state transition published to Kafka | ✅ deployed |

## Rules engine consumers (illustrative rulesets)

| Ruleset name | Version | Attached to | Sample rule (pseudo-JSON) |
|---|---|---|---|
| `pmjay.preauth.v1` | 1 | zorbit-app-hi_preauth | `{ "all": [{ "fact": "package_code", "operator": "inList", "value": "PMJAY_ICU_CARDIAC" }] } → { "event": "preauth.auto-approve", "params": { "limit": 50000 } }` |
| `pmjay.intake.completeness.v1` | 1 | zorbit-app-hi_claim_initiation | `{ "all": [{ "fact": "has_aadhaar", "operator": "equal", "value": true }, { "fact": "discharge_summary_uploaded", "operator": "equal", "value": true }] } → { "event": "intake.ready-for-verification" }` |
| `pmjay.adjudication.auto.v1` | 1 | hi_claim_adjudication_workflow | `{ "all": [{ "fact": "kyc_status", "operator": "equal", "value": "verified" }, { "fact": "package_amount_within_limit", "operator": "equal", "value": true }, { "fact": "mandatory_docs_count", "operator": "greaterThan", "value": 4 }] } → { "event": "adjudication.auto-approve", "params": { "confidence": 0.95 } }` |
| `pmjay.recon.match.v1` | 1 | hi_claim_payment_recon | `{ "any": [{ "fact": "utr_match", "operator": "equal", "value": true }, { "fact": "amount_match_and_date_within_3d", "operator": "equal", "value": true }] } → { "event": "recon.auto-match" }` |

Rules authored by: analysts (via admin UI form_builder rules editor) + AI-assisted (send natural-language policy → JSON rules, human-approve).

## BPMN processes (minimum set for PMJAY E2E)

1. **BP-PMJAY-PREAUTH** — pre-auth lifecycle (4 UserTasks, 2 XOR gateways)
2. **BP-PMJAY-INTAKE** — FNOL + evidence upload (3 UserTasks, 1 parallel gateway for doc-upload + pixel-processing)
3. **BP-PMJAY-ADJUDICATION** — auto-adjudicate first, manual fallback (2 service tasks + 2 UserTasks + 1 XOR)
4. **BP-PMJAY-PAYMENT** — NHA disbursement (1 service task to payment_gateway + 1 UserTask for hospital confirmation)
5. **BP-PMJAY-RECON** — daily recon cycle (timer start event + 3 stages)

All BPMN XMLs seeded into `zorbit-pfs-workflow_engine` under `seeds/bpmn/pmjay/`.

## PII fields + mandatory tokenisation (pii_vault)

| Field | Why PII |
|---|---|
| claimant_name | India DPDPA + PMJAY rules |
| claimant_aadhaar | Aadhaar Act |
| claimant_pan | Tax-ID |
| claimant_phone | DPDPA |
| claimant_bank_account | financial |
| claimant_email | DPDPA |
| claimant_address | DPDPA |
| hospital_registration_no | business-confidential, not PII but masked in non-adjudicator views |

Each stored ONLY as a PII token (e.g., `PII-A1B2`) in the operational DB. Raw resolved on-the-fly per role:
- `pmjay_adjudicator` → full
- `pmjay_auditor` → masked (xxxx-xxxx-1234)
- everyone else → tokenised (`PII-A1B2`)

## Pixel (document digitisation) — integration contract

- Input: any file (PDF, JPEG, PNG) via `POST /api/pixel/api/v1/O/:org/documents` (multipart)
- Output: structured JSON extracting claim-relevant fields (provider name, IPD admission date, diagnosis, package code, discharge summary bullet points, line items, totals, signatures/stamps confidence)
- Async; emits `platform.document.digitized` event with doc hash + structured fields
- Downstream: adjudication service subscribes, auto-fills fields + runs rules
- Pixel port: **in progress — awaiting owner source repo pointer**

## Unified console integration

- "Claims" left-nav section with children: Pre-auth, Intake, Adjudication (my queue / team queue / auto-approved), Payments, Reconciliation
- Each page uses `<DataTable>` for lists + `<FormRenderer>` for forms (both from `zorbit-sdk-react`)
- Rules authoring page (admin-only) under Settings > Rules Library
- Workflow visualizer embedded from `zorbit-pfs-workflow_engine`'s bpmn-js designer

## Test data — seed plan

Use `zorbit-pfs-seeder` to create:
- 1 O-PMJAY tenant
- 10 hospitals (each empanelled with ~20 PMJAY packages)
- 100 claimants with full PII (tokenised)
- 50 ongoing claims in various lifecycle stages
- 3 BPMN-instance samples per stage

Seed idempotent; `POST /api/pfs-seeder/api/v1/G/seed/pmjay` flushes + reseeds.

## Risk / fallbacks

- **Pixel delay**: if owner's source repo isn't available in time, fall back to a "manual data entry" intake form. Pre-auth + adjudication still work on typed fields.
- **Rules engine complexity** (owner's note): if json-rules-engine nesting gets unwieldy for adjudication, build an "independent claim adjudication engine" that's PMJAY-specific. Keep it as `zorbit-app-hi_claim_adjudication_workflow/src/lib/engine/` for later merge into `pfs-rules_engine`.
- **BPMN complexity** for auto-adjudication: keep first implementation as 90% auto, 10% manual; add sub-process drill-down only if adjudicator feedback demands it.

## Open questions for owner (when available)

1. PMJAY rule files — is there an authoritative source (e.g., NHA policy PDF) or do we encode from sample cases?
2. Claim dispute channel — through our module or external portal?
3. Provider-side UI — same console or separate hospital portal?
4. Analytics dashboards — inside console (via `pfs-analytics_reporting`) or exported to external BI (Superset as `zs-superset`)?
