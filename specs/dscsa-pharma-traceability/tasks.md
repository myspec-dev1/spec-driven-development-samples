# Implementation Tasks: DSCSA Pharmaceutical Serialization & Traceability Platform

## Phase 1: Serialization & Identifier Foundations
- [ ] **TSK-DSC-01**: Build GTIN/NDC master data service and randomized serial pool allocator with per-GTIN uniqueness guarantees.
- [ ] **TSK-DSC-02**: Implement GS1 codec converting DataMatrix element strings (AI 01/21/10/17), EPC URIs (SGTIN, SSCC), and GS1 Digital Link, with check-digit and character-set validation.
- [ ] **TSK-DSC-03**: Create `serialized_units` registry with `ltree` hierarchy paths and lifecycle state CHECK constraints.

## Phase 2: EPCIS 2.0 Capture & Aggregation Engine
- [ ] **TSK-DSC-04**: Implement EPCIS 2.0 REST capture/query gateway (JSON-LD + XML) with CBV validation and event-hash de-duplication.
- [ ] **TSK-DSC-05**: Build per-serial lifecycle state machine consuming Kafka events partitioned by EPC, rejecting out-of-order or post-decommission events.
- [ ] **TSK-DSC-06**: Implement packing/unpacking with the aggregation conservation check and single-parent disjointness rule.
- [ ] **TSK-DSC-07**: Implement ship/receive inference on intact containers and the `INFERENCE_BROKEN` fallback to unit-level scans.
- [ ] **TSK-DSC-08**: Build hash-chained append-only event journal with EPCIS `errorDeclaration` corrections.

## Phase 3: Trading Partner Exchange & Saleable Returns
- [ ] **TSK-DSC-09**: Build trading partner registry with exemption-class dates and Authorized Trading Partner checks (state license / FDA registration, 24-hour cache).
- [ ] **TSK-DSC-10**: Implement TI/TS generation and exchange as EPCIS documents, plus receiving reconciliation of scanned SSCCs/SGTINs against received TI.
- [ ] **TSK-DSC-11**: Implement VRS requester with GS1 lookup directory routing and restock gate for saleable returns.
- [ ] **TSK-DSC-12**: Implement VRS responder for manufacturer tenants backed by a Redis active-serial cache.

## Phase 4: Investigations, FDA Reporting & EU FMD
- [ ] **TSK-DSC-13**: Build quarantine service that blocks shipment of all affected serials and subtrees within 60 seconds of a suspect flag.
- [ ] **TSK-DSC-14**: Build investigation workbench with Part 11-style e-signatures and FDA Form 3911 prefill driven by a 24-hour countdown watchdog.
- [ ] **TSK-DSC-15**: Implement tracing-request responder assembling TI/TS and full event history per product identifier, with 1-business-day SLA tracking.
- [ ] **TSK-DSC-16**: Build EU FMD connector for EU Hub pack upload, verification, decommissioning, and 10-day status reversal.
- [ ] **TSK-DSC-17**: Implement 6-year WORM evidence archive (S3 Object Lock + Parquet) with chain-hash verification jobs.

## Phase 5: Compliance Benchmarking & Validation
- [ ] **TSK-DSC-18**: Load-test the capture gateway at 50,000 EPCIS events/second for 1 hour with zero lost or duplicated events.
- [ ] **TSK-DSC-19**: Benchmark VRS responder at 2,000 requests/second; verify p99 latency $\le 1\text{ second}$.
- [ ] **TSK-DSC-20**: Verify aggregation conservation across 10,000 synthetic pallets (10,000 units each, 100M units total) with random unpack/repack, and confirm pallet inference in $\le 500\text{ms}$.
- [ ] **TSK-DSC-21**: Run a mock FDA tracing request for 500 product identifiers spanning 6 years of data; verify a complete response in $\le 15\text{ minutes}$.
