# Implementation Tasks: Carbon Removal MRV & Credit Registry Platform

## Phase 1: Project Registration & MRV Evidence Ingestion
- [ ] **TSK-CDR-01**: Build versioned methodology catalog (biochar, DACCS, BECCS, ERW) with CRCF activity type, equations, and pinned emission factor sets.
- [ ] **TSK-CDR-02**: Implement PDD intake with PostGIS project polygons, storage site IDs, crediting periods, and additionality/safeguard evidence checklists.
- [ ] **TSK-CDR-03**: Build MRV ingestion gateway for SCADA telemetry, lab assay certificates, and storage injection reports with signature validation.
- [ ] **TSK-CDR-04**: Implement WORM evidence store (S3 Object Lock) with SHA-256 content addressing linked to monitoring periods.

## Phase 2: Net Removal Quantification & Buffer Pool
- [ ] **TSK-CDR-05**: Implement net removal engine (gross - baseline - LCA - leakage) with deficit carry-forward across periods.
- [ ] **TSK-CDR-06**: Build Monte Carlo uncertainty module (10,000 draws, reproducible seeds) and tiered 90% CI discount rules.
- [ ] **TSK-CDR-07**: Implement biochar persistence model (H/C$_{\text{org}} \le 0.7$, soil temperature, 100-year horizon).
- [ ] **TSK-CDR-08**: Build non-permanence risk scoring tool and buffer pool account with reversal cancellation workflow.

## Phase 3: VVB Validation & Verification Workflow
- [ ] **TSK-CDR-09**: Implement VVB accreditation registry with scope/expiry checks, conflict-of-interest blocking, and 6-verification rotation rule.
- [ ] **TSK-CDR-10**: Build Temporal workflows for CAR / CL / FAR findings with issuance gating while any CAR or CL is open.
- [ ] **TSK-CDR-11**: Implement X.509 qualified-signature verification statements with assurance level and $\le 5\%$ materiality threshold.

## Phase 4: Serialized Unit Ledger & Double-Counting Controls
- [ ] **TSK-CDR-12**: Implement dual-approval serialized issuance with non-overlapping serial ranges (GiST exclusion constraints).
- [ ] **TSK-CDR-13**: Build atomic range split for transfers, retirement with beneficiary certificates, and irreversible cancellation.
- [ ] **TSK-CDR-14**: Implement SHA-256 hash-chained event journal with daily Merkle roots anchored via RFC 3161 timestamping.
- [ ] **TSK-CDR-15**: Build cross-registry overlap detection (geolocation, storage site, feedstock batch) and Article 6 LoA / corresponding adjustment tracking.

## Phase 5: Public Registry API
- [ ] **TSK-CDR-16**: Build read-only public REST API (projects, issuances, retirements, buffer balances) with cursor pagination and CDN caching.
- [ ] **TSK-CDR-17**: Implement serial number lookup resolving range, vintage, state, beneficiary, and authorization status.

## Phase 6: Integrity Benchmarking & Validation
- [ ] **TSK-CDR-18**: Load-test ledger at 2,000 transfers/retirements per second; verify p99 commit latency $\le 150\text{ms}$ and zero range overlaps.
- [ ] **TSK-CDR-19**: Run 1,000,000 randomized transfer/split/retire/cancel operations; verify conservation $\sum \text{Issued} = \sum \text{Active} + \sum \text{Retired} + \sum \text{Cancelled} + \sum \text{Buffer}$ holds after every commit.
- [ ] **TSK-CDR-20**: Verify Monte Carlo quantification of a 50,000-sample period completes in $\le 60\text{ seconds}$ with bit-identical results on re-run.
- [ ] **TSK-CDR-21**: Inject 500 simulated reversal events; confirm buffer cancellations complete within 5 business days and NDC/CORSIA retirements without `ca_status = APPLIED` are rejected 100% of the time.
