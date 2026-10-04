# Implementation Tasks: DORA ICT Risk & Operational Resilience Platform

## Phase 1: ICT Inventory & Dependency Graph
- [ ] **TSK-DRA-01**: Build ICT asset, business-function, and CIF registry with CMDB import (ServiceNow CMDB, CSV).
- [ ] **TSK-DRA-02**: Implement PostgreSQL + Apache AGE dependency graph (function → asset → ICT service → provider → subcontractor) with transitive CIF queries.
- [ ] **TSK-DRA-03**: Implement 5×5 inherent/residual risk scoring with mandatory treatment decisions for residual $R \ge 15$.

## Phase 2: Incident Classification & Reporting
- [ ] **TSK-DRA-04**: Build ITSM/SIEM intake webhooks (ServiceNow, Jira SM, Sentinel, Splunk) with immutable `detected_at` / `aware_at` capture.
- [ ] **TSK-DRA-05**: Implement versioned RTS classification engine with per-criterion evaluation persisted to `criteria_evaluation`.
- [ ] **TSK-DRA-06**: Implement recurring-incident aggregator (same root cause, $\ge 2$ occurrences within 6 months).
- [ ] **TSK-DRA-07**: Build Temporal deadline watchdog for initial (4h / 24h), intermediate (72h), and final (1 month) reports with 50/75/90% escalations.
- [ ] **TSK-DRA-08**: Implement report generators for initial, intermediate, and final templates plus client-notification drafts, with four-eyes sign-off.

## Phase 3: Register of Information & xBRL-CSV
- [ ] **TSK-DRA-09**: Model ITS Register of Information templates (`B_01.01`–`B_07.01`, `B_99.01`) at entity and consolidated level.
- [ ] **TSK-DRA-10**: Implement LEI MOD 97-10 checksum and GLEIF API validation with EUID fallback.
- [ ] **TSK-DRA-11**: Build xBRL-CSV report package builder (`reportPackage.json`, `parameters.csv`, table CSVs) and run ESA validation rules via Arelle.

## Phase 4: Concentration Risk, Testing & Exit Strategies
- [ ] **TSK-DRA-12**: Implement HHI concentration analytics per ICT service type with subcontractor-chain (hidden concentration) detection.
- [ ] **TSK-DRA-13**: Build annual resilience testing programme tracker with findings, owners, and remediation SLAs.
- [ ] **TSK-DRA-14**: Implement TLPT (TIBER-EU) lifecycle tracker: preparation, threat intelligence, red team ($\ge 12$ weeks), purple team, remediation, attestation.
- [ ] **TSK-DRA-15**: Build exit-plan registry enforcing a tested plan for every CIF-supporting contract.

## Phase 5: Evidence Integrity & Security
- [ ] **TSK-DRA-16**: Implement SHA-256 hash chaining for incidents, reports, and register snapshots with S3 Object Lock (compliance mode, 5-year retention).
- [ ] **TSK-DRA-17**: Integrate SSO with FIDO2/WebAuthn MFA and four-eyes approval for report submission and register release.

## Phase 6: Benchmarking & Validation
- [ ] **TSK-DRA-18**: Replay 10,000 synthetic incidents against a golden RTS test set; verify 100% classification agreement and $\le 200\text{ms}$ p99 latency.
- [ ] **TSK-DRA-19**: Chaos-test the deadline watchdog (region failover, pod kills) across 1,000 concurrent major incidents; verify zero missed deadlines and RTO $\le 15\text{ minutes}$.
- [ ] **TSK-DRA-20**: Export a 50,000-arrangement register to xBRL-CSV in $\le 2\text{ minutes}$ with zero ESA validation errors.
