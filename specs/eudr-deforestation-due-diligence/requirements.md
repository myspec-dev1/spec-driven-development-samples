# Requirements Specification: EUDR Deforestation Due-Diligence & Traceability Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **EUDR Compliance Officer (ACT-CMP)**: Owns the due diligence system, approves risk conclusions, and signs DDS submissions on behalf of the operator.
- **Sourcing & Supplier Manager (ACT-SRC)**: Onboards upstream suppliers, cooperatives, and traders, and collects plot data and legality documents.
- **Field Mapping Agent (ACT-FLD)**: Walks plot boundaries with GNSS-enabled mobile devices and captures farmer, plot, and harvest data offline.
- **GIS & Remote-Sensing Analyst (ACT-GIS)**: Reviews satellite deforestation flags, overlays imagery, and resolves `MANUAL_REVIEW` plots.
- **Downstream Trader / Operator (ACT-DWN)**: Receives goods with upstream DDS reference and verification numbers and references them in its own DDS.
- **Competent Authority Inspector (ACT-INS)**: Requests due diligence evidence during checks and receives exported audit packages.
- **TRACES Integration Daemon (ACT-TRC)**: Submits, amends, and retrieves DDS records via the EU Information System API.

---

## 2. Functional Requirements

### 2.1 Plot Geolocation Capture & Validation (FR-GEO)
- **FR-GEO-01 (Point vs Polygon Rule)**: The system MUST accept a single point (latitude/longitude) for plots of $\le 4\text{ ha}$ and MUST require a closed polygon for plots $> 4\text{ ha}$ (for all commodities except cattle). For cattle, the system MUST record the geolocation of every establishment where the animals were kept.
- **FR-GEO-02 (Coordinate Format)**: All geometries MUST be stored as WGS84 (EPSG:4326) GeoJSON (RFC 7946) with $\ge 6$ decimal places; inputs with fewer decimals MUST be flagged `LOW_PRECISION` and blocked from DDS use.
- **FR-GEO-03 (Topology Validation)**: Polygons MUST pass OGC validity checks (closed rings, no self-intersection, counter-clockwise exterior per RFC 7946) and MUST NOT overlap another supplier's plot by $> 5\%$ of area without a resolution note.
- **FR-GEO-04 (Plausibility Checks)**: The system MUST reject points falling in water bodies, urban footprints, or outside the declared country of production, and MUST flag declared yields exceeding $\text{area}_{\text{ha}} \times \text{max yield}_{\text{commodity,country}}$ (e.g. cocoa $> 1{,}500\text{ kg/ha/yr}$).
- **FR-GEO-05 (Bulk Import)**: The system MUST import GeoJSON, KML, Shapefile, and CSV (lat/lon) files of up to 100,000 plots per upload with row-level error reports.

### 2.2 Satellite Deforestation Screening (FR-SAT)
- **FR-SAT-01 (Baseline Forest Mask)**: The system MUST determine whether each plot was forest on 31 Dec 2020 using JRC Global Forest Cover 2020 (10 m) as primary baseline, with Hansen Global Forest Change (30 m) tree-cover loss years 2021+ as secondary evidence.
- **FR-SAT-02 (Post-Cut-Off Loss Detection)**: The system MUST compute deforested area inside each plot (plus a 30 m buffer around points) and classify:
  $$\text{status} = \begin{cases} \texttt{CLEAR} & A_{\text{loss} > 2020} = 0 \\ \texttt{MANUAL\_REVIEW} & 0 < A_{\text{loss} > 2020} \le 0.5\text{ ha} \\ \texttt{FLAGGED} & A_{\text{loss} > 2020} > 0.5\text{ ha} \end{cases}$$
- **FR-SAT-03 (Near-Real-Time Alerts)**: The system MUST re-screen active plots weekly against GLAD-L/GLAD-S2 and RADD alert layers and notify ACT-GIS within 24 hours of a new alert.
- **FR-SAT-04 (Evidence Snapshots)**: Each screening result MUST persist dataset versions, tile IDs, and before/after imagery thumbnails for audit.

### 2.3 Risk Assessment & Mitigation (FR-RSK)
- **FR-RSK-01 (Country Benchmarking)**: The system MUST apply the Commission's country benchmarking (low / standard / high risk) per country or sub-national region, version-tracked by effective date.
- **FR-RSK-02 (Multi-Criteria Risk Score)**: The system MUST compute a documented score combining country risk, deforestation status, supplier history, legality evidence (land tenure, labour, tax, FPIC of indigenous peoples), and supply-chain complexity (number of tiers, blending events).
- **FR-RSK-03 (Simplified Due Diligence)**: For products sourced exclusively from low-risk countries, the system MUST allow simplified due diligence (no formal risk assessment) while still enforcing FR-GEO and FR-SAT, unless information indicates circumvention.
- **FR-RSK-04 (Mitigation Workflow)**: Non-negligible risk MUST open a mitigation case (document request, independent audit, field verification) that blocks DDS submission until closed with dual approval by ACT-CMP.

### 2.4 Chain of Custody & Traceability (FR-COC)
- **FR-COC-01 (Custody Models)**: The system MUST support `SEGREGATED` (single-origin lots) and `CONTROLLED_BLEND` (lots mixed only from fully geolocated, `CLEAR` plots, all declared in the DDS). Credit-based mass balance with unknown-origin inputs MUST be rejected.
- **FR-COC-02 (Volume Conservation)**: For each transformation (e.g. cocoa beans `1801` → cocoa butter `1804`), the system MUST enforce $Q_{\text{out}} \le \sum Q_{\text{in}} \times \text{conversion factor}_{\text{HS}}$ with tolerances $\le 2\%$.
- **FR-COC-03 (Multi-Tier Linking)**: The system MUST link lots across $\ge 6$ supplier tiers (farm → cooperative → exporter → processor → trader → operator) and resolve every output lot to its plot set in $\le 2\text{ s}$.
- **FR-COC-04 (HS Code Scope)**: The system MUST maintain the Annex I HS/CN code scope (e.g. `0901` coffee, `1201` soya, `1511` palm oil, `4001` rubber, `4403` wood, `0102` cattle) and flag shipments with in-scope codes lacking a DDS.

### 2.5 DDS Submission & TRACES Integration (FR-DDS)
- **FR-DDS-01 (DDS Assembly)**: The system MUST assemble a DDS containing operator identity, HS code, product description, quantity (net mass in kg, and supplementary units where applicable), country of production, plot geolocations, and the compliance declaration.
- **FR-DDS-02 (TRACES Submission)**: The system MUST submit DDSs to the EU Information System API and persist the returned **reference number** and **verification number**, with status tracking (`SUBMITTED`, `AVAILABLE`, `REJECTED`, `WITHDRAWN`).
- **FR-DDS-03 (Upstream DDS Referencing)**: Downstream operators MUST be able to reference upstream DDSs by reference + verification number instead of re-uploading geolocation, with the system validating both numbers before use.
- **FR-DDS-04 (Customs Linking)**: The DDS reference number MUST be exportable to the customs declaration workflow before goods are released for free circulation or exported.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance & Scale (NFR-PERF)
- **NFR-PERF-01 (Screening Throughput)**: The satellite screening engine MUST process $\ge 1{,}000{,}000$ plots per hour on cached Cloud-Optimized GeoTIFF tiles.
- **NFR-PERF-02 (DDS Turnaround)**: DDS assembly for a shipment covering 50,000 plots MUST complete in $\le 60\text{ s}$ (excluding TRACES API latency).

### 3.2 Security, Integrity & Data Protection (NFR-SEC)
- **NFR-SEC-01 (Tamper-Evident Records)**: All due diligence records MUST be hash-chained (SHA-256) and retained for $\ge 5$ years with WORM storage.
- **NFR-SEC-02 (Farmer Personal Data)**: Smallholder names, IDs, and precise coordinates are personal data under GDPR; access MUST be role-scoped, encrypted at rest (AES-256), and shared with downstream parties only via DDS references.

### 3.3 Reliability & Resilience (NFR-REL)
- **NFR-REL-01 (Offline Capture)**: The mobile app MUST capture and queue $\ge 500$ plots offline for 30 days with conflict-free sync.
- **NFR-REL-02 (Idempotent Submission)**: TRACES submissions MUST be idempotent via client-side submission keys; retries after timeouts MUST NOT create duplicate DDSs.

---

## 4. Due-Diligence & DDS Submission Pipeline

```
[Field Mapping App / Bulk Import]          [Supplier Lots & Shipments (ERP)]
   (GeoJSON / KML / SHP, WGS84)               (HS codes, quantities, tiers)
                 |                                         |
                 v                                         v
     [Geometry Validation (FR-GEO)]          [Chain-of-Custody Ledger (FR-COC)]
                 |                                         |
                 v                                         |
   [Satellite Screening vs 31-12-2020]                     |
     (JRC GFC2020 / Hansen / RADD)                         |
                 |                                         |
       +---------+----------+                              |
       |         |          |                              |
   [CLEAR]  [MANUAL_REVIEW] [FLAGGED] --> [Excluded]       |
       |         |                                         |
       |    [GIS Analyst]                                  |
       |         |                                         |
       +----+----+                                         |
            |                                              |
            +--------------------+-------------------------+
                                 |
                                 v
             [Risk Assessment + Country Benchmark (FR-RSK)]
                                 |
                  +--------------+--------------+
                  |                             |
          [Negligible Risk]           [Non-Negligible Risk]
                  |                             |
                  |                    [Mitigation Case] --> (re-assess)
                  v
       [DDS Assembly & Sign-off] ---> [EU TRACES API] ---> [Reference # + Verification #]
                                                                    |
                                                                    v
                                              [Customs Declaration / Downstream DDS Link]
                                                                    |
                                                                    v
                                                  [WORM Evidence Archive (5 years)]
```
