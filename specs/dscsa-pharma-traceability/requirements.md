# Requirements Specification: DSCSA Pharmaceutical Serialization & Traceability Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **Serialization Manager (ACT-SER)**: Allocates serial number ranges per GTIN, approves packaging-line commissioning batches, and manages master data (NDC, GTIN, company prefix).
- **Trading Partner Compliance Officer (ACT-TPC)**: Onboards trading partners, verifies state licenses and FDA registrations, and approves TI/TS exchange connections.
- **Returns Processor (ACT-RET)**: Scans saleable returns at the distributor dock and triggers product identifier verification before restock.
- **Quality & Regulatory Investigator (ACT-QRI)**: Quarantines suspect product, runs investigations, files FDA Form 3911, and answers FDA tracing requests.
- **Packaging Line & WMS Integrations (ACT-INT)**: Machine actors (Level 3 site servers, WMS, ERP) that post EPCIS events and shipment data.
- **External Trading Partner System (ACT-EXT)**: Upstream/downstream partners and their VRS responders, EPCIS repositories, and the EU EMVS hub.

---

## 2. Functional Requirements

### 2.1 Serialization & Product Identifiers (FR-SER)
- **FR-SER-01 (Product Identifier Composition)**: The system MUST model each package-level product identifier as GTIN-14 (embedding the 10-digit NDC) + serial number (up to 20 alphanumeric characters) + lot number + expiration date, encoded in a GS1 DataMatrix with Application Identifiers `(01)` GTIN, `(21)` serial, `(10)` lot, and `(17)` expiry `YYMMDD`.
- **FR-SER-02 (Serial Allocation)**: The system MUST allocate non-sequential (randomized) serials per GTIN with guaranteed uniqueness, with a guessing probability of $\le 10^{-4}$ for EU FMD packs.
- **FR-SER-03 (EPC Encoding)**: The system MUST convert between GS1 element strings, EPC Pure Identity URIs (`urn:epc:id:sgtin:...`, `urn:epc:id:sscc:...`), and GS1 Digital Link URIs without loss.
- **FR-SER-04 (Barcode Validation)**: The system MUST reject scans with an invalid GTIN or SSCC check digit, an expiry date that is not a valid calendar date, or a serial outside the GS1 AI 82 character set.

### 2.2 EPCIS 2.0 Event Capture (FR-EPC)
- **FR-EPC-01 (Event Types & Vocabulary)**: The system MUST capture EPCIS 2.0 `ObjectEvent`, `AggregationEvent`, and `TransactionEvent` (JSON-LD and XML) using CBV business steps `commissioning`, `packing`, `unpacking`, `shipping`, `receiving`, `decommissioning`, and `destroying`, with matching dispositions (`active`, `in_transit`, `in_progress`, `inactive`, `destroyed`).
- **FR-EPC-02 (Lifecycle State Machine)**: The system MUST validate each event against the serial's current state; for example, a `shipping` event for a serial that was never commissioned or is already decommissioned MUST be rejected and flagged.
- **FR-EPC-03 (Idempotent Capture)**: Each event MUST carry an `eventID` (hash-based per the EPCIS 2.0 event hash algorithm when the sender does not supply one); resubmitting the same event MUST NOT create duplicates.
- **FR-EPC-04 (Error Declaration)**: Corrections MUST use EPCIS `errorDeclaration` with `correctiveEventIDs`; the original event remains queryable.

### 2.3 Aggregation Hierarchy (FR-AGG)
- **FR-AGG-01 (Multi-Level Packing)**: The system MUST record unit → bundle → case → pallet hierarchies, where cases and pallets are identified by SSCC-18 or SGTIN.
- **FR-AGG-02 (Conservation Check)**: On every `packing`/`unpacking` event, the system MUST verify that each child has exactly one parent and that the expanded unit set of a parent equals the union of its children's unit sets.
- **FR-AGG-03 (Inference on Ship/Receive)**: When a pallet SSCC is shipped or received with an intact hierarchy, the system MUST infer the event for all $N$ descendant units without per-unit scans.
- **FR-AGG-04 (Hierarchy Break)**: A damaged case, partial unpack, or count mismatch at receiving MUST mark the container `INFERENCE_BROKEN` and require unit-level scans before onward shipment.

### 2.4 Transaction Data Exchange & Trading Partners (FR-TRX)
- **FR-TRX-01 (TI/TS Generation)**: For each change of ownership, the system MUST produce Transaction Information (product name, strength, dosage form, NDC, container size, number of containers, lot, product identifiers, transaction date, shipment date, seller and buyer names and addresses) and a Transaction Statement, sent as EPCIS 2.0 per the industry DSCSA EPCIS implementation guidelines.
- **FR-TRX-02 (Advance Delivery)**: TI/TS MUST be delivered to the buyer before or at the time of physical shipment; receiving MUST reconcile scanned SSCCs/SGTINs against the received TI.
- **FR-TRX-03 (Authorized Trading Partner Check)**: Before any send, accept, or restock, the system MUST confirm the counterparty holds a valid state license or FDA registration (cached for $\le 24\text{ hours}$, re-checked on expiry).
- **FR-TRX-04 (Exemption Scheduling)**: The system MUST store each partner's DSCSA exemption class and enforce electronic exchange from the applicable date (manufacturers/repackagers May 27, 2025; wholesale distributors Aug 27, 2025; dispensers with 26+ full-time pharmacists/technicians Nov 27, 2025; small dispensers Nov 27, 2026).

### 2.5 Saleable Returns Verification (FR-VRS)
- **FR-VRS-01 (VRS Requester)**: For each saleable return, the system MUST send a verification request (GTIN, serial, lot, expiry) to the manufacturer's VRS responder resolved through the GS1 lookup directory, using the GS1 Lightweight Messaging Standard.
- **FR-VRS-02 (VRS Responder)**: For manufacturer tenants, the system MUST answer incoming verification requests with a `verified`/`not verified` result and alert codes.
- **FR-VRS-03 (Restock Gate)**: Returned units MUST NOT be restocked unless the response is `verified`; `not verified` or timeout MUST move the unit to quarantine and open an investigation.

### 2.6 Suspect & Illegitimate Product Investigation (FR-INV)
- **FR-INV-01 (Quarantine on Suspicion)**: When a product is flagged suspect (failed verification, duplicate serial, decommissioned serial in commerce, or partner notice), the system MUST quarantine every affected serial within $\le 60\text{ seconds}$ and block its shipping.
- **FR-INV-02 (Investigation Workflow)**: The system MUST guide the investigator through manufacturer verification, TI/TS review, and disposition, recording each step with timestamps and user identity.
- **FR-INV-03 (Form 3911 Clock)**: When an investigator determines product is illegitimate, the system MUST prefill FDA Form 3911, notify FDA and affected trading partners within $\le 24\text{ hours}$, and track termination of the notification after FDA consultation.
- **FR-INV-04 (Tracing Requests)**: For FDA or partner requests in a recall or investigation, the system MUST assemble TI/TS and the full event history for the requested product identifiers within 1 business day ($\le 48\text{ hours}$), with an internal target of $\le 15\text{ minutes}$.

### 2.7 EU FMD Secondary Market (FR-FMD)
- **FR-FMD-01 (EMVS Upload)**: The system MUST upload pack data (product code, serial, batch, expiry, and national reimbursement number where required) to the EU Hub before release for sale.
- **FR-FMD-02 (Status Changes)**: The system MUST support verification and decommissioning (supplied, destroyed, exported, sample, stolen) and reverting a status within 10 days when permitted.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance & Throughput (NFR-PERF)
- **NFR-PERF-01 (Event Ingestion)**: The event repository MUST sustain 50,000 EPCIS events/second, including commissioning bursts of 600 units/minute per packaging line across 200 lines.
- **NFR-PERF-02 (VRS Latency)**: VRS responses MUST be returned in $\le 1\text{ second}$ at p99 under 2,000 requests/second.
- **NFR-PERF-03 (Pallet Inference)**: Inferring a ship event for a 5-level pallet with 10,000 units MUST complete in $\le 500\text{ms}$.

### 3.2 Security, Integrity & Compliance (NFR-SEC)
- **NFR-SEC-01 (Tamper Evidence)**: Every stored event MUST be chained by SHA-256 hash per tenant; any break in the chain MUST raise a critical alert.
- **NFR-SEC-02 (Partner Authentication)**: Partner exchange MUST use mutual TLS 1.3 and OAuth 2.0 client credentials; data shared with a partner MUST be limited to transactions involving that partner.
- **NFR-SEC-03 (Electronic Records)**: Investigator decisions and Form 3911 submissions MUST carry 21 CFR Part 11-style audit trails and electronic signatures.

### 3.3 Reliability & Retention (NFR-REL)
- **NFR-REL-01 (Durability)**: Accepted events MUST have RPO = 0 (synchronous multi-AZ commit) and RTO $\le 15\text{ minutes}$.
- **NFR-REL-02 (6-Year Retention)**: All traceability records MUST be retained for at least 6 years in WORM storage and remain queryable within the tracing-request SLA.

---

## 4. Saleable Return & Suspect Product Flow

```
[Returned Unit Scanned at Dock (ACT-RET)]
     (GS1 DataMatrix: 01 / 21 / 10 / 17)
                 |
                 v
     [Parse & Validate Identifier] --- invalid barcode ---> [Quarantine + Open Investigation]
                 |
                 v
     [Local Lifecycle Check: commissioned? shipped to us? not decommissioned?]
                 |
         +-------+--------+
         |                |
       [OK]          [Conflict] ---------------------------> [Quarantine + Open Investigation]
         |                                                              |
         v                                                              v
 [VRS Request to Manufacturer]                          [Manufacturer Verification + TI/TS Review]
         |                                                              |
   +-----+--------------+                                    +----------+-----------+
   |                    |                                    |                      |
[Verified]   [Not Verified / Timeout] ----------------> [Cleared]          [Illegitimate]
   |                                                         |                      |
   v                                                         v                      v
[ATP Check on Returning Partner]                   [Release from Quarantine]  [FDA Form 3911 <= 24h]
   |                                                                        [Notify Trading Partners]
   v                                                                                |
[Restock: ObjectEvent receiving / sellable]                                         v
                                                                     [Disposition: destroying event]
```
