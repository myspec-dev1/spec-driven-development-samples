# Solution Architecture: FHIR Payer Interoperability & Electronic Prior Authorization Platform

## 1. System Architecture Overview

The platform is architected for regulatory-clock precision, standards conformance, and consent-aware data release:
1. **SMART Authorization Server (Keycloak / OIDC)**: Issues SMART App Launch and Backend Services tokens, enforces v2 granular scopes, and binds member consent and provider attribution claims into access tokens.
2. **FHIR R4 Data Platform (HAPI FHIR JPA / PostgreSQL)**: Serves US Core, CARIN Blue Button, PDex, and ATR resources for Patient Access, Provider Access, and Payer-to-Payer APIs, with Bulk Data `$export` workers writing NDJSON to encrypted object storage.
3. **Da Vinci CRD/DTR Services (TypeScript / Node.js + CQL Engine)**: Hosts CDS Hooks endpoints and a coverage rules engine, and serves `Questionnaire` + CQL libraries for DTR SMART apps.
4. **PAS Orchestrator & Decision Clock (TypeScript / Temporal)**: Validates PAS Bundles, persists requests, schedules deadline timers as durable workflows, and routes work to UM reviewer queues.
5. **X12 278 Gateway (Java / Smooks EDI)**: Bidirectionally maps PAS Bundles to X12 278 (005010X217) for the legacy UM system and clearinghouses.
6. **Metrics & Audit Pipeline (Kafka / ClickHouse)**: Streams decision events and `AuditEvent` records to compute public prior authorization metrics and regulator audit extracts.

```
+-----------------------------------------------------------------------------------------+
|                                     EXTERNAL CLIENTS                                    |
|                                                                                         |
|   [Provider EHRs (CDS Hooks / SMART)]   [Member Apps (SMART)]   [Other Payers (Bulk)]   |
+-----------------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                        API GATEWAY + SMART AUTHORIZATION SERVER                         |
|                                                                                         |
|  [OAuth 2.0 / OIDC] ---> [Scope + Consent Enforcement] ---> [Attribution Check (ATR)]   |
+-----------------------------------------------------------------------------------------+
                     |                                         |
                     v                                         v
+---------------------------------------+   +---------------------------------------------+
|     DA VINCI PRIOR AUTH SERVICES      |   |          FHIR R4 DATA PLATFORM (HAPI)       |
|  - CRD CDS Hooks + Coverage Rules     |   |  - Patient Access (CARIN BB / US Core)      |
|  - DTR Questionnaire + CQL Package    |   |  - Provider Access (Group/$export)          |
|  - PAS Claim/$submit, $inquire        |   |  - Payer-to-Payer (opt-in, 5-yr lookback)   |
+---------------------------------------+   +---------------------------------------------+
                     |                                         ^
                     v                                         |
+---------------------------------------+                      |
|   PAS ORCHESTRATOR & DECISION CLOCK   | ---------------------+  (PA status <= 1 bus. day)
|  - Temporal deadline workflows        |
|  - UM reviewer queues                 | <---> [X12 278 Gateway] <---> [Legacy UM System]
+---------------------------------------+
                     |  (Kafka: pa.decision.events)
                     v
+-----------------------------------------------------------------------------------------+
|          METRICS & AUDIT PIPELINE: AuditEvent Journal -> Annual Public PA Report        |
+-----------------------------------------------------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Prior Authorization Request & Decision Clock (`pa_requests`)
```sql
CREATE TABLE pa_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bundle_identifier VARCHAR(128) NOT NULL UNIQUE, -- PAS Bundle.identifier (idempotency key)
    member_id VARCHAR(64) NOT NULL,
    requesting_provider_npi CHAR(10) NOT NULL CHECK (requesting_provider_npi ~ '^[0-9]{10}$'),
    item_category VARCHAR(32) NOT NULL CHECK (item_category IN ('MEDICAL_SERVICE', 'DME', 'IMAGING', 'PROCEDURE', 'HOME_HEALTH', 'INPATIENT')),
    priority VARCHAR(16) NOT NULL CHECK (priority IN ('EXPEDITED', 'STANDARD')),
    channel VARCHAR(16) NOT NULL CHECK (channel IN ('FHIR_PAS', 'X12_278', 'PORTAL', 'FAX')),
    received_at TIMESTAMPTZ NOT NULL,
    decision_due_at TIMESTAMPTZ NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'PENDED' CHECK (status IN ('PENDED', 'APPROVED', 'MODIFIED', 'DENIED', 'CANCELLED')),
    decided_at TIMESTAMPTZ,
    denial_reason_code VARCHAR(32),
    denial_reason_text TEXT,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CHECK (decision_due_at = received_at + CASE priority WHEN 'EXPEDITED' THEN INTERVAL '72 hours' ELSE INTERVAL '7 days' END),
    CHECK (status NOT IN ('DENIED', 'MODIFIED') OR (denial_reason_code IS NOT NULL AND length(denial_reason_text) > 0)),
    CHECK (decided_at IS NULL OR decided_at >= received_at)
);
```

### 2.2 Member Data-Sharing Consent Ledger (`member_consents`)
```sql
CREATE TABLE member_consents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id VARCHAR(64) NOT NULL,
    api_scope VARCHAR(24) NOT NULL CHECK (api_scope IN ('PAYER_TO_PAYER', 'PROVIDER_ACCESS')),
    decision VARCHAR(8) NOT NULL CHECK (decision IN ('OPT_IN', 'OPT_OUT')),
    consent_version VARCHAR(16) NOT NULL,
    capture_channel VARCHAR(16) NOT NULL CHECK (capture_channel IN ('MEMBER_APP', 'PORTAL', 'PHONE', 'PAPER', 'ENROLLMENT')),
    previous_payer_id VARCHAR(64), -- required for PAYER_TO_PAYER opt-in
    effective_at TIMESTAMPTZ NOT NULL,
    recorded_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CHECK (api_scope <> 'PAYER_TO_PAYER' OR decision <> 'OPT_IN' OR previous_payer_id IS NOT NULL)
);
CREATE INDEX idx_member_consents_latest ON member_consents (member_id, api_scope, effective_at DESC);
```

## 3. PAS Decision Clock & ClaimResponse Builder
```typescript
const EXPEDITED_MS = 72 * 60 * 60 * 1000;
const STANDARD_MS = 7 * 24 * 60 * 60 * 1000;
const X12_306 = 'https://codesystem.x12.org/005010/306';
const REVIEW_ACTION_URL = 'http://hl7.org/fhir/us/davinci-pas/StructureDefinition/extension-reviewAction';

type Outcome = 'APPROVED' | 'MODIFIED' | 'DENIED' | 'PENDED';
const REVIEW_ACTION_CODE: Record<Outcome, string> = { APPROVED: 'A1', MODIFIED: 'A6', DENIED: 'A3', PENDED: 'A4' };

export function computeDecisionDeadline(receivedAt: Date, priority: 'EXPEDITED' | 'STANDARD'): Date {
  // Clock is anchored to authoritative receipt time; calendar days, not business days.
  return new Date(receivedAt.getTime() + (priority === 'EXPEDITED' ? EXPEDITED_MS : STANDARD_MS));
}

export function buildClaimResponse(req: PaRequest, decision: ItemDecision[], now: Date): fhir4.ClaimResponse {
  if (now.getTime() > req.decisionDueAt.getTime() && decision.some((d) => d.outcome === 'PENDED')) {
    clockBreachCounter.inc({ priority: req.priority }); // surfaced in metrics + supervisor paging
  }

  return {
    resourceType: 'ClaimResponse',
    status: 'active',
    type: { coding: [{ system: 'http://terminology.hl7.org/CodeSystem/claim-type', code: req.claimType }] },
    use: 'preauthorization',
    patient: { reference: `Patient/${req.memberId}` },
    created: now.toISOString(),
    insurer: { reference: `Organization/${req.payerOrgId}` },
    request: { identifier: { value: req.bundleIdentifier } },
    outcome: decision.every((d) => d.outcome !== 'PENDED') ? 'complete' : 'queued',
    preAuthRef: req.authorizationNumber,
    item: decision.map((d) => {
      if ((d.outcome === 'DENIED' || d.outcome === 'MODIFIED') && !d.reasonCode) {
        throw new Error(`Item ${d.sequence}: a specific denial reason is mandatory before finalizing.`);
      }
      return {
        itemSequence: d.sequence,
        adjudication: [{ category: { coding: [{ code: 'submitted' }] } }],
        extension: [
          {
            url: REVIEW_ACTION_URL,
            extension: [
              { url: 'code', valueCodeableConcept: { coding: [{ system: X12_306, code: REVIEW_ACTION_CODE[d.outcome] }] } },
              ...(d.reasonCode
                ? [{ url: 'reasonCode', valueCodeableConcept: { coding: [{ code: d.reasonCode }], text: d.reasonText } }]
                : []),
            ],
          },
        ],
      };
    }),
  };
}
```
