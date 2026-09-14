# Requirements Specification: Modular Cloud ERP & Shop Floor MES Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **Chief Financial Officer / Controller (ACT-FIN)**: Manages fiscal calendars, multi-currency consolidations, tax filings, and GL audits.
- **Production Planner / Master Scheduler (ACT-PLN)**: Generates Material Requirements Planning (MRP) runs, capacity schedules, and work orders.
- **Shop Floor Machine Operator (ACT-OPR)**: Interacts with ruggedized tablet terminals, clocks into jobs, reports scrap, and scans lot barcodes.
- **Quality Engineer (ACT-QA)**: Sets up incoming/in-process inspection checkpoints, flags Quarantine lots, and handles CAPA workflows.
- **Plant Maintenance Engineer (ACT-MNT)**: Monitors machine telemetry, vibration alerts, preventive maintenance work orders, and spare parts.

---

## 2. Functional Requirements

### 2.1 Double-Entry Core Ledger & Financial Management (FR-FIN)
- **FR-FIN-01 (Multi-Entity General Ledger)**: The system MUST maintain a segmented Chart of Accounts (COA) supporting multi-company, multi-currency, and intercompany elimination ledgers.
- **FR-FIN-02 (Atomic Transaction Posting)**: All sub-ledgers (AR, AP, Inventory, Payroll) MUST post to the General Ledger atomically in balanced debit/credit journals. An un-balanced journal entry ($\sum \text{Debit} \neq \sum \text{Credit}$) MUST be rejected with error `ERR_LEDGER_UNBALANCED`.
- **FR-FIN-03 (Automated Multi-Jurisdiction Tax Engine)**: The system MUST calculate real-time VAT, GST, and US Sales tax rates based on origin/destination addresses via integration with tax tables or external providers (Avalara/TaxJar).
- **FR-FIN-04 (Fiscal Period Closing & Year-End Rollover)**: The system MUST allow locking prior fiscal periods (Soft Close / Hard Close) preventing back-dated postings without administrative override.

### 2.2 Multi-Level Bill of Materials (BOM) & Engineering Change Orders (FR-BOM)
- **FR-BOM-01 (Hierarchical Recursive BOM Structure)**: The system MUST support n-level recursive assembly structures containing sub-assemblies, raw materials, phantom assemblies, and outsourced manufacturing operations.
- **FR-BOM-02 (Cost Rollup Engine)**: The system MUST calculate standard, average, and FIFO unit costs rolling up raw material costs, direct machine run labor, and overhead absorption rates across all nested levels in $\le 5\text{ seconds}$ for a 1,000-part BOM.
- **FR-BOM-03 (Engineering Change Order - ECO Workflow)**: ECOs MUST follow a formal approval state machine (`Draft -> Review -> Approved -> Effective -> Obsolete`). When effective, all pending MRP runs MUST switch to the new BOM revision while existing released work orders remain locked.

### 2.3 Shop Floor Control, Work Orders & Machine Execution (FR-MES)
- **FR-MES-01 (Work Order Generation & Reservation)**: When a sales order or safety stock threshold triggers demand, the system MUST create Work Orders, allocating inventory lots and checking work center routing capacities.
- **FR-MES-02 (Operator Terminal Interface)**: Shop floor touchscreen UI MUST support one-tap Operator Clock-In/Clock-Out, Job Start, Pause, and Completion with barcode/QR scanning for batch/lot IDs.
- **FR-MES-03 (Scrap & Downtime Reason Tracking)**: Operators MUST be required to select a validated downtime reason code (e.g. `TOOL_BREAKAGE`, `RAW_MATERIAL_DEFECT`, `NO_OPERATOR`) whenever a workstation halts for $> 3\text{ minutes}$.
- **FR-MES-04 (Overall Equipment Effectiveness - OEE Engine)**: The system MUST compute OEE in real-time for each machine work center:
  $$\text{OEE} = \text{Availability} \times \text{Performance} \times \text{Quality}$$
  updating live dashboard charts every 5 seconds.

### 2.4 IoT Industrial Telemetry & Preventative Maintenance (FR-IOT)
- **FR-IOT-01 (OPC-UA / MQTT Telemetry Ingestion)**: The system MUST ingest high-frequency telemetry (temperature, current, vibration, spindle RPM, part counter pulses) via an edge MQTT broker.
- **FR-IOT-02 (Threshold & Predictive Alerts)**: When machine operating metrics exceed $\pm 3\sigma$ from baseline running limits, the system MUST automatically trigger an urgent Maintenance Work Order and notify the plant technician.

### 2.5 Quality Assurance & Lot Genealogy (FR-QA)
- **FR-QA-01 (Full Backward/Forward Lot Genealogy)**: Given any finished product serial number, the system MUST trace back to every component batch, supplier PO, operator ID, inspection test result, and machine center within $\le 2\text{ seconds}$.
- **FR-QA-02 (Quarantine & Disposition Enforcement)**: If a lot fails an in-process QA sample inspection, the system MUST immediately flag the lot as `QUARANTINED`, preventing shipping or downstream consumption.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance & Scalability (NFR-PERF)
- **NFR-PERF-01 (Financial Post Latency)**: Journal entry commit latency MUST be $\le 25\text{ms}$ at p99.
- **NFR-PERF-02 (Shop Floor Response Time)**: Barcode scan-to-screen verification on operator tablets MUST be $\le 150\text{ms}$ over local plant Wi-Fi.
- **NFR-PERF-03 (IoT Ingestion Throughput)**: Telemetry broker MUST ingest at least 50,000 MQTT metric messages/sec with end-to-end processing latency $\le 250\text{ms}$.

### 3.2 Reliability, Availability & Edge Autonomy (NFR-REL)
- **NFR-REL-01 (Offline Resilience)**: Local edge gateway appliances at manufacturing facilities MUST continue functioning and buffering production logs locally for at least 48 continuous hours during WAN link disconnections.
- **NFR-REL-02 (High Availability)**: Cloud ERP core MUST provide 99.95% uptime SLA with RPO = 0 (zero lost transactions) and RTO $\le 30\text{ seconds}$.

### 3.3 Security & Compliance (NFR-SEC)
- **NFR-SEC-01 (Audit Trail)**: Strict compliance with SOC 1 / SOC 2 Type II controls and ISO 9001 Section 7.5 (Documented Information).
- **NFR-SEC-02 (Role-Based Access Control)**: Enforce separation of duties (SoD) ensuring the user who creates a Vendor Purchase Order cannot approve payment disbursements for the same vendor.

---

## 4. Manufacturing State Machine & Core Workflow

```
[Sales / MRP Demand] -> [Release Work Order] -> [Pick & Stage Raw Lots]
                                                        |
                                                        v
                                             [Operator Start Job]
                                                        |
                                      +-----------------+-----------------+
                                      |                                   |
                             [Machine Cycle / IoT]               [Part Quality Check]
                                      |                                   |
                             [Record Downtime]                  +---------+---------+
                                                                |                   |
                                                            [Pass QA]           [Fail QA]
                                                                |                   |
                                                        [WIP / Finish]        [Quarantine]
                                                                |
                                                     [Post GL & Cost Rollup]
```
