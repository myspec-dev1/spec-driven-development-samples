# Implementation Tasks: FHIR Payer Interoperability & Electronic Prior Authorization Platform

## Phase 1: FHIR Platform, Identity & Consent Foundations
- [ ] **TSK-PAS-01**: Deploy HAPI FHIR R4 (4.0.1) server with US Core, CARIN Blue Button, PDex, and Da Vinci ATR profiles loaded and validated.
- [ ] **TSK-PAS-02**: Configure SMART App Launch (v2 granular scopes), SMART Backend Services (`private_key_jwt`), and OIDC identity on the authorization server.
- [ ] **TSK-PAS-03**: Implement `member_consents` ledger with Payer-to-Payer opt-in and Provider Access opt-out capture across app, portal, and phone channels.
- [ ] **TSK-PAS-04**: Build append-only `AuditEvent` journal for every PHI disclosure with 6-year retention.

## Phase 2: Patient Access, Provider Access & Payer-to-Payer APIs
- [ ] **TSK-PAS-05**: Extend Patient Access API with non-drug prior authorization status, decision dates, and denial reasons published within 1 business day of change.
- [ ] **TSK-PAS-06**: Implement provider attribution (`Group` maintenance) and Bulk Data `Group/[id]/$export` NDJSON workers for Provider Access.
- [ ] **TSK-PAS-07**: Build Payer-to-Payer request/response flow with opt-in gating, 5-year lookback, and quarterly concurrent-coverage exchange.

## Phase 3: Coverage Requirements Discovery & DTR
- [ ] **TSK-PAS-08**: Implement CRD CDS Services for `order-select`, `order-sign`, and `appointment-book` with `coverage-information` extension output.
- [ ] **TSK-PAS-09**: Author coverage rules and DTR `Questionnaire` + CQL libraries for the top 50 PA-required service codes.
- [ ] **TSK-PAS-10**: Implement `Questionnaire/$questionnaire-package` and adaptive `$next-question` with EHR prepopulation via SMART launch.

## Phase 4: PAS Submission, X12 278 & Decision Clock
- [ ] **TSK-PAS-11**: Implement PAS `Claim/$submit` with Bundle validation, idempotency on `Bundle.identifier`, and synchronous `ClaimResponse`.
- [ ] **TSK-PAS-12**: Build bidirectional PAS ↔ X12 278 (005010X217) translator with round-trip conformance tests.
- [ ] **TSK-PAS-13**: Implement Temporal decision-clock workflows (72h expedited / 7d standard) with supervisor paging at 80% of window.
- [ ] **TSK-PAS-14**: Build UM reviewer queue with mandatory specific denial reason and Medical Director sign-off on adverse determinations.
- [ ] **TSK-PAS-15**: Implement `Claim/$inquire` and FHIR Subscriptions notifying EHRs of pended-to-final transitions.

## Phase 5: Metrics Reporting & Compliance Validation
- [ ] **TSK-PAS-16**: Build ClickHouse metrics pipeline computing approval, denial, appeal-overturn, extension rates, and average/median decision times per priority.
- [ ] **TSK-PAS-17**: Generate annual machine-readable and accessible HTML public metrics report for 31 March publication.

## Phase 6: Performance Benchmarking & Conformance Validation
- [ ] **TSK-PAS-18**: Load-test CRD at 500 hook calls/second; verify $\le 1\text{ s}$ p95 and $\le 2\text{ s}$ p99 latency.
- [ ] **TSK-PAS-19**: Run Bulk `$export` for a 50,000-member attribution group; verify completion in $\le 30\text{ minutes}$.
- [ ] **TSK-PAS-20**: Replay 100,000 synthetic PA requests through the decision clock; verify 0 missed deadlines and 100% of denials carry a specific reason.
- [ ] **TSK-PAS-21**: Pass Inferno / Touchstone conformance suites for US Core, SMART App Launch, Bulk Data, CRD, DTR, and PAS with zero failed required tests.
