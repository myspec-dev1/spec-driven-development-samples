# Implementation Tasks: EU Digital Battery Passport Platform

## Phase 1: Passport Identity & GS1 Digital Link Resolver
- [ ] **TSK-BAT-01**: Implement passport minting service (GTIN + serial, UUIDv7 key) with category and $> 2\text{ kWh}$ industrial scope validation.
- [ ] **TSK-BAT-02**: Build ISO/IEC 18004 QR code generator encoding GS1 Digital Link URIs with GTIN check-digit validation.
- [ ] **TSK-BAT-03**: Implement Go resolver with content negotiation (HTML, JSON-LD, `application/linkset+json`) and edge snapshot caching.
- [ ] **TSK-BAT-04**: Create `battery_passports` schema with partial unique index enforcing one active passport per physical battery.

## Phase 2: Annex XIII Attribute Catalogue & Access Tiers
- [ ] **TSK-BAT-05**: Build versioned catalogue of ~70 attributes with mandatory tier tags (`PUBLIC`, `LEGITIMATE_INTEREST`, `AUTHORITY`).
- [ ] **TSK-BAT-06**: Implement default-deny tier filter in the resolver and the passport API.
- [ ] **TSK-BAT-07**: Integrate OpenID4VP role credential verification for repairers, remanufacturers, recyclers, and authorities.
- [ ] **TSK-BAT-08**: Implement access audit log recording caller, tier, purpose, and battery ID for every non-public read.

## Phase 3: Verifiable Supplier Claims
- [ ] **TSK-BAT-09**: Build Rust credential verifier for VC 2.0 Data Integrity (`ecdsa-rdfc-2019`) and JOSE/COSE ES256 proofs.
- [ ] **TSK-BAT-10**: Implement `did:web` / `did:key` resolution and Bitstring Status List revocation polling with passport flagging.
- [ ] **TSK-BAT-11**: Map carbon footprint, recycled content (Co/Li/Ni/Pb), and due diligence credentials to catalogue attributes with 2031 threshold checks.

## Phase 4: BMS Telemetry & Lifecycle Succession
- [ ] **TSK-BAT-12**: Deploy EMQX MQTT 5 ingestion into Kafka and TimescaleDB with per-device signature verification.
- [ ] **TSK-BAT-13**: Implement SoH plausibility filter (reject $> 2\%$ unexplained increases) and daily dynamic-attribute digest.
- [ ] **TSK-BAT-14**: Build lifecycle state machine (`ORIGINAL → REPURPOSED/REMANUFACTURED → WASTE → RECYCLED`) rejecting backward transitions.
- [ ] **TSK-BAT-15**: Implement passport succession: issue new passport, set `predecessor_id`, mark old as `SUPERSEDED`, transfer operator responsibility.

## Phase 5: Federation, Backup & Signing
- [ ] **TSK-BAT-16**: Implement HSM-backed ES256 signing of every passport version snapshot with SHA-256 content hash.
- [ ] **TSK-BAT-17**: Build backup replicator to an independent third-party host with S3 Object Lock and content-addressed snapshots.

## Phase 6: Benchmarking & Compliance Validation
- [ ] **TSK-BAT-18**: Load test resolver at 5,000 requests/second; verify p95 $\le 150\text{ms}$ and p99 $\le 400\text{ms}$.
- [ ] **TSK-BAT-19**: Replay 50,000 telemetry readings/second across 20,000,000 simulated passports with zero dropped readings.
- [ ] **TSK-BAT-20**: Run tier leakage test suite over 100% of catalogue attributes; verify zero non-public attributes in `PUBLIC` responses.
- [ ] **TSK-BAT-21**: Simulate primary operator outage; verify all passports resolve from backup host within RPO $\le 5\text{ min}$.
