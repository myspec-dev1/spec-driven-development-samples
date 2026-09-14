# Implementation Tasks: Modular Cloud ERP & Shop Floor MES Platform

## Phase 1: Core Double-Entry General Ledger & Chart of Accounts
- [ ] **TSK-ERP-01**: Implement PostgreSQL schema for `gl_journal_entries` and `gl_lines` with transactional balancing triggers.
- [ ] **TSK-ERP-02**: Build Chart of Accounts (COA) tree hierarchy service with segment validation (entity, department, account).
- [ ] **TSK-ERP-03**: Create sub-ledger posting adapters for Accounts Payable (AP) bills and Accounts Receivable (AR) invoices.
- [ ] **TSK-ERP-04**: Implement automated fiscal period closing locks and foreign exchange currency revaluation jobs.

## Phase 2: Multi-Level BOM & Material Requirements Planning (MRP)
- [ ] **TSK-ERP-05**: Develop recursive multi-level BOM engine supporting components, scrap factors, and phantom assemblies.
- [ ] **TSK-ERP-06**: Build Cost Rollup engine to compute standard, FIFO, and moving average costs across nested BOM trees.
- [ ] **TSK-ERP-07**: Implement Engineering Change Order (ECO) lifecycle state machine with deviation approvals.
- [ ] **TSK-ERP-08**: Build forward-scheduling and backward-scheduling MRP algorithm with lead time calculation.

## Phase 3: Shop Floor MES Execution & Operator Terminals
- [ ] **TSK-ERP-09**: Create responsive touch-optimized Web / PWA terminal UI for shop floor operators.
- [ ] **TSK-ERP-10**: Implement 2D barcode / QR code scanning for lot serialization, material verification, and traveler sheets.
- [ ] **TSK-ERP-11**: Build Operator Clock-in/Clock-out and job tracking time logger with labor absorption calculations.
- [ ] **TSK-ERP-12**: Implement scrap, rework, and machine downtime reporting modal with mandatory categorized reason codes.

## Phase 4: Industrial IoT Telemetry, Edge Gateway & OEE
- [ ] **TSK-ERP-13**: Deploy EMQX / Mosquitto MQTT broker on edge gateway hardware with TLS client certificates.
- [ ] **TSK-ERP-14**: Build OPC-UA to MQTT industrial translation daemon for legacy CNC machines and PLCs.
- [ ] **TSK-ERP-15**: Implement TimescaleDB continuous aggregate hypertable for real-time sensor ingestion.
- [ ] **TSK-ERP-16**: Build real-time OEE computation engine and WebSocket streaming pipeline for plant dashboards.

## Phase 5: Quality Management & End-to-End Lot Genealogy
- [ ] **TSK-ERP-17**: Implement inbound receiving inspection workflows with AQL (Acceptable Quality Limit) sampling tables.
- [ ] **TSK-ERP-18**: Build Quarantine disposition state machine blocking inventory movement of non-conforming items.
- [ ] **TSK-ERP-19**: Develop bi-directional lot genealogy query engine tracing finished serial numbers to raw supplier lots in $< 2\text{s}$.
- [ ] **TSK-ERP-20**: Implement Corrective and Preventive Action (CAPA) tracking module with automated escalations.

## Phase 6: Edge Synchronization & Load Validation
- [ ] **TSK-ERP-21**: Test offline-mode edge resilience by severing WAN link during 1,000 simulated production cycles; verify zero data loss upon reconnect.
- [ ] **TSK-ERP-22**: Benchmark concurrent ledger commits under 10,000 simulated work order completions; ensure p99 commit latency $\le 25\text{ms}$.
