# Requirements Specification: Warehouse Management System (WMS) & Multimodal Freight Logistics Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **Warehouse General Manager (ACT-WGM)**: Oversees facility throughput, labor allocation, dock door scheduling, and inventory accuracy.
- **Forklift Driver / Order Picker (ACT-PCK)**: Executes directed pick-lists via rugged handheld RF terminal scanners or voice-directed headsets.
- **Inbound Receiving Clerk (ACT-REC)**: Unloads containers, inspects packing slips, validates ASN (Advanced Shipping Notice), and prints pallet license plate labels (LPN).
- **Freight Dispatcher / Logistics Coordinator (ACT-DSP)**: Tenders loads, tracks ocean/air containers, and monitors carrier milestones.
- **Customs Compliance Specialist (ACT-CST)**: Validates HS tariff classifications, rules of origin, and files electronic import entries.

---

## 2. Functional Requirements

### 2.1 Inbound Receiving, Putaway & License Plate Numbering (FR-INB)
- **FR-INB-01 (Advanced Shipping Notice - ASN Ingestion)**: The system MUST ingest EDI 856 / DESADV notices from suppliers and match expected items, quantities, and PO numbers upon dock arrival.
- **FR-INB-02 (Pallet License Plate Number - LPN Generation)**: The system MUST assign a unique GS1-128 compliant License Plate Number (LPN) barcode to each received pallet containing SKU, batch, expiry date, and weight.
- **FR-INB-03 (Directed Putaway Algorithm)**: The putaway engine MUST calculate the optimal storage bin based on velocity ranking (ABC analysis), height/weight constraints, hazardous material compatibility, and current zone congestion in $\le 200\text{ms}$.

### 2.2 Order Fulfillment, Picking & Wave Management (FR-OUT)
- **FR-OUT-01 (Intelligent Wave & Batch Release)**: The system MUST aggregate pending sales orders into optimized picking waves grouped by carrier pickup cutoff times, shipping priority, and warehouse zone.
- **FR-OUT-02 (Pick Route Optimization)**: The system MUST calculate the shortest travel path across aisles (Traveling Salesperson Problem heuristic) minimizing dead-heading for order pickers.
- **FR-OUT-03 (Pick-to-Light / Barcode Confirmation)**: Pickers MUST scan both the source bin barcode and the product SKU barcode to confirm each pick. Mismatches MUST trigger an audible alert and block confirmation until corrected.
- **FR-OUT-04 (Automated Packing & Dimensioning)**: The packing station MUST weigh and scan package dimensions (cubing), select optimal carton sizes, generate carrier compliant UCC-128 shipping labels, and output commercial packing slips in $\le 800\text{ms}$.

### 2.3 Multimodal Freight Forwarding & Leg State Machine (FR-FRT)
- **FR-FRT-01 (Multi-Leg Freight Itinerary)**: The system MUST model complex multimodal journeys comprising sequential legs (First Mile Drayage, Air Freight, Ocean Cargo, Rail Intermodal, Final Mile LTL/FTL).
- **FR-FRT-02 (Electronic Carrier Load Tendering)**: The system MUST generate and dispatch EDI 204 / IFTMIN load tenders to contracted carriers and receive EDI 990 / APERAK confirmation acceptances.
- **FR-FRT-03 (Milestone Event Ingestion & Geofencing)**: The system MUST ingest real-time GPS telemetry from trucks, AIS satellite tracks for ocean vessels, and EDI 214 / IFTSTA status updates, detecting route deviations or delays exceeding 60 minutes.

### 2.4 Demurrage & Detention Countdown Watchdog (FR-DMR)
- **FR-DMR-01 (Port Terminal Free-Time Clock)**: When an ocean container discharges onto the marine terminal (`Vessel Discharged` milestone), the system MUST automatically start the Free-Time countdown clock (e.g. 5 days free).
- **FR-DMR-02 (Calculated Financial Exposure)**: For every day a container remains past free-time, the system MUST accrue demurrage penalties based on tier rate schedules (e.g., Days 1-4: $150/day; Days 5+: $300/day).
- **FR-DMR-03 (Automated Dispatch Escalation)**: When free-time reaches $\le 24\text{ hours}$ remaining, the system MUST prioritize the container for emergency drayage dispatch and alert the logistics manager.

### 2.5 Automated Customs Classification & Filing (FR-CST)
- **FR-CST-01 (Harmonized System - HS Code Catalog)**: The system MUST maintain global 6-digit WCO HS codes and national 10-digit tariff schedules (US HTS, EU TARIC).
- **FR-CST-02 (Electronic Customs Declaration Export)**: The system MUST generate standard CBP ACE XML / CDS European customs declaration files with automated duty and VAT calculation based on country of origin.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Handheld Terminal Latency & Ergonomics (NFR-PERF)
- **NFR-PERF-01 (Scan-to-Acknowledge)**: Handheld RF terminal barcode scan response time MUST be $\le 100\text{ms}$ over warehouse 802.11ax networks.
- **NFR-PERF-02 (Picking Wave Computation)**: Wave planning algorithm optimizing 5,000 orders across 200,000 square feet MUST complete calculation in $\le 10\text{ seconds}$.

### 3.2 High Availability & Disaster Recovery (NFR-REL)
- **NFR-REL-01 (Zero Downtime Maintenance)**: Fulfillment centers operate 24/7/365; database schema migrations and software deployments MUST execute without pausing pick-pack operations.
- **NFR-REL-02 (High Availability)**: Platform SLA of 99.99% availability; RPO = 0, RTO $\le 30\text{ seconds}$.

### 3.3 Protocol Interoperability & Security (NFR-SEC)
- **NFR-SEC-01 (AS2 / SFTP EDI Encryption)**: All EDI transmissions via AS2 MUST enforce S/MIME encryption (AES-256) and SHA-256 digital signature receipts (MDN).

---

## 4. Multimodal Shipment State Machine

```
[Booking Confirmed] -> [Carrier Tendered (EDI 204)] -> [Drayage Inbound]
                                                               |
                                                               v
                                                    [Port Gate-In (Terminal)]
                                                               |
                                                               v
                                                    [Vessel Loaded (EDI 304)]
                                                               |
                                                      (Ocean Transit / AIS)
                                                               |
                                                               v
                                                [Vessel Discharged (Demurrage Clock Starts!)]
                                                               |
                                           +-------------------+-------------------+
                                           |                                       |
                                [Customs Cleared]                          [Customs Inspection Hold]
                                           |                                       |
                                           v                                       v
                                [Drayage Outbound]                         [Escalated Clearance]
                                           |
                                           v
                                [Delivery Destination (EDI 214)]
```
