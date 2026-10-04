# Requirements Specification: EU Digital Battery Passport Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **Battery Manufacturer / Economic Operator (ACT-MFR)**: Places batteries on the EU market, mints passports, and is legally responsible for passport data until responsibility transfers.
- **Upstream Supplier (ACT-SUP)**: Cell, cathode, and raw material suppliers issuing signed claims for carbon footprint, recycled content, and due diligence.
- **Vehicle OEM / BMS Gateway (ACT-BMS)**: Pushes dynamic telemetry (SoH, cycle count, energy throughput) from the in-field Battery Management System.
- **Repairer / Second-Life Operator (ACT-RSL)**: Interested person with legitimate interest who repairs, repurposes, or remanufactures batteries and triggers passport succession.
- **Recycler / Waste Handler (ACT-RCY)**: Interested person who declares a battery as waste and accesses dismantling and material composition data.
- **Market Surveillance Authority / Commission (ACT-AUT)**: Notified bodies, national authorities, and the European Commission with full read access for compliance checks.
- **Public Consumer (ACT-PUB)**: Anonymous end user scanning the QR code to view publicly accessible attributes.

---

## 2. Functional Requirements

### 2.1 Passport Identity & Data Carrier (FR-IDN)
- **FR-IDN-01 (Unique Passport Identifier)**: The system MUST mint a globally unique, never-reused passport identifier per battery unit, composed of a GS1 GTIN (model) plus a serial number (`AI 01` + `AI 21`), and a UUIDv7 internal key.
- **FR-IDN-02 (QR Data Carrier)**: The system MUST generate an ISO/IEC 18004 QR code (error correction level M or higher) encoding a GS1 Digital Link URI, e.g. `https://id.example-battery.eu/01/04012345000010/21/SN8842117`.
- **FR-IDN-03 (Resolver Content Negotiation)**: The resolver MUST return HTML for browsers, JSON-LD for `Accept: application/ld+json`, and a GS1 linkset for `Accept: application/linkset+json`.
- **FR-IDN-04 (Scope Gate)**: The system MUST require passports only for categories `EV`, `LMT`, and `INDUSTRIAL` with rated energy $E_{\text{rated}} > 2\text{ kWh}$, and MUST reject issuance for out-of-scope categories with an explicit validation error.

### 2.2 Data Model & Access Tiers (FR-ACC)
- **FR-ACC-01 (Annex XIII Attribute Catalogue)**: The system MUST maintain a versioned catalogue of ~70 attributes grouped into: general & identity, compliance & labels, carbon footprint, supply chain due diligence, materials & composition, circularity & recycled content, and performance & durability.
- **FR-ACC-02 (Per-Attribute Tier Tagging)**: Each attribute MUST carry exactly one tier tag (`PUBLIC`, `LEGITIMATE_INTEREST`, `AUTHORITY`); untagged attributes MUST be excluded from every response.
- **FR-ACC-03 (Tier Resolution)**: Visible attributes for a caller $c$ MUST be computed as:
  $$\text{Visible}(c) = \bigcup_{t \in \text{Tiers}(c)} \{ a \in \text{Catalogue} : a.\text{tier} = t \}$$
  where `PUBLIC` $\subseteq$ every caller's tiers and `AUTHORITY` callers receive all three tiers.
- **FR-ACC-04 (Legitimate Interest Credentials)**: Repairers, remanufacturers, and recyclers MUST authenticate with an OpenID4VP presentation of a role credential (e.g. `RecyclerCredential`) issued by a trusted accreditation authority; each grant is logged with purpose and battery ID.

### 2.3 Supplier Claims & Verifiable Credentials (FR-VCR)
- **FR-VCR-01 (Signed Claim Ingestion)**: The system MUST accept supplier claims only as W3C Verifiable Credentials 2.0 secured with Data Integrity (`ecdsa-rdfc-2019`) or JOSE/COSE (ES256) proofs.
- **FR-VCR-02 (Issuer Resolution & Revocation)**: The system MUST resolve issuer DIDs (`did:web`, `did:key`) and check Bitstring Status List revocation before accepting a claim; revoked claims MUST flag the affected passports within 15 minutes.
- **FR-VCR-03 (Carbon Footprint Declaration)**: The system MUST store total lifecycle carbon footprint in $\text{kg CO}_2\text{e per kWh}$ of total energy over service life, the lifecycle-stage breakdown, the performance class, and a link to the public study.
- **FR-VCR-04 (Recycled Content Shares)**: The system MUST store pre-consumer and post-consumer recycled shares for cobalt, lithium, nickel, and lead, and MUST flag shortfalls against the minimum shares applicable from 18 August 2031 (Co $16\%$, Li $6\%$, Ni $6\%$, Pb $85\%$).

### 2.4 Dynamic Performance Telemetry (FR-TEL)
- **FR-TEL-01 (BMS Telemetry Ingestion)**: The system MUST ingest signed telemetry batches (SoH, remaining capacity, cycle count, energy throughput, internal resistance increase, negative events such as deep discharge or thermal excursions) via MQTT 5 or HTTPS.
- **FR-TEL-02 (State of Health Computation)**: Capacity-based SoH MUST be recorded as:
  $$\text{SoH}_{\text{cap}} = \frac{C_{\text{remaining}}}{C_{\text{rated}}} \times 100\%$$
  and readings MUST be rejected if $\text{SoH}$ increases by $> 2\%$ versus the prior accepted reading without a remanufacturing event.
- **FR-TEL-03 (Update Cadence Throttle)**: The passport-visible dynamic values MUST be refreshed at most once per 24 hours per battery; raw readings are retained in the telemetry store.

### 2.5 Lifecycle & Passport Succession (FR-LCY)
- **FR-LCY-01 (Status State Machine)**: The system MUST enforce transitions `ORIGINAL → {REPURPOSED, REMANUFACTURED, WASTE}`, `{REPURPOSED, REMANUFACTURED} → WASTE`, and `WASTE → RECYCLED`; any other transition MUST be rejected.
- **FR-LCY-02 (New Passport on Repurposing)**: When ACT-RSL repurposes or remanufactures a battery, the system MUST issue a new passport, set `predecessor_id`, freeze the old passport as `SUPERSEDED`, and transfer operator responsibility to the new economic operator.
- **FR-LCY-03 (Waste Declaration)**: When ACT-RCY declares a battery as waste, the system MUST lock all static attributes and expose dismantling and safety information to the recycler tier.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance & Scale (NFR-PERF)
- **NFR-PERF-01 (Resolver Latency)**: QR resolution MUST complete in $\le 150\text{ms}$ at p95 and $\le 400\text{ms}$ at p99 for 5,000 requests/second.
- **NFR-PERF-02 (Telemetry Throughput)**: Telemetry ingestion MUST sustain 50,000 readings/second across a fleet of 20,000,000 passports.

### 3.2 Security & Trust (NFR-SEC)
- **NFR-SEC-01 (Signed Passport Snapshots)**: Every published passport version MUST be signed by the economic operator's key (ES256, HSM-backed, FIPS 140-3 Level 3) with a SHA-256 content hash.
- **NFR-SEC-02 (Tier Leakage Prevention)**: Automated tests MUST prove zero `LEGITIMATE_INTEREST` or `AUTHORITY` attributes appear in `PUBLIC` responses across 100% of catalogue attributes.

### 3.3 Availability & Durability (NFR-REL)
- **NFR-REL-01 (Resolver Availability)**: The public resolver MUST achieve 99.95% monthly availability.
- **NFR-REL-02 (Independent Backup Copy)**: Each passport version MUST be replicated to a third-party backup host within 5 minutes (RPO $\le 5\text{ min}$) and remain resolvable if the primary operator ceases trading.

---

## 4. Passport Resolution & Lifecycle Pipeline

```
[Supplier VCs (CO2e, Recycled %, DD)]   [Manufacturer ERP / MES]   [BMS Telemetry (MQTT 5)]
               |                                  |                          |
               v                                  v                          v
     [VC Verifier: DID + Status List]   [Passport Minting (GTIN+SN)]  [SoH Plausibility Filter]
               |                                  |                          |
               +------------------+---------------+--------------------------+
                                  |
                                  v
                    [Passport Store (versioned, signed)]
                                  |
              +-------------------+-------------------+
              |                                       |
              v                                       v
  [QR / GS1 Digital Link Resolver]          [Lifecycle State Machine]
              |                                       |
   +----------+-----------+              +------------+-------------+
   |          |           |              |                          |
[PUBLIC] [LEGIT. INT.] [AUTHORITY]  [REPURPOSED / REMANUFACTURED]  [WASTE -> RECYCLED]
                                         |
                                         v
                              [New Passport + predecessor link]
```
