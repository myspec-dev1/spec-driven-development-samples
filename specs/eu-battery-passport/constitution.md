# Constitution: EU Digital Battery Passport Platform

## 1. Purpose & Mission
The EU Digital Battery Passport Platform issues, hosts, and maintains the mandatory electronic battery passport required by Regulation (EU) 2023/1542 (Article 77 and Annex XIII) for every electric vehicle (EV) battery, light means of transport (LMT) battery, and industrial battery above 2 kWh placed on the EU market from 18 February 2027. It binds each physical battery to a unique passport identifier, resolves a QR data carrier to a role-filtered passport view, ingests signed supplier claims (carbon footprint, recycled content, due diligence), streams dynamic State of Health (SoH) telemetry from Battery Management Systems (BMS), and tracks the battery through repurposing, remanufacturing, and end-of-life recycling.

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: One Battery, One Identifier, One Active Passport
- Every physical battery unit is bound to exactly one globally unique passport identifier for its current life; identifiers are never reused, even after the battery is recycled.
- At any instant, at most one passport per physical battery may hold an active status:
  $$\forall b \in \text{Batteries}: \quad \left| \{ p \in \text{Passports} : p.\text{battery} = b \land p.\text{state} = \text{ACTIVE} \} \right| \le 1$$
- When a battery is repurposed or remanufactured, a new passport is issued and cryptographically linked to its predecessor; the predecessor passport state is frozen as `SUPERSEDED`, never deleted.

### Tenet 2: Access Tier Enforcement by Default-Deny
- Every attribute is tagged with exactly one Annex XIII access tier: `PUBLIC`, `LEGITIMATE_INTEREST` (repairers, remanufacturers, second-life operators, recyclers), or `AUTHORITY` (notified bodies, market surveillance authorities, European Commission).
- An attribute without a tier tag is never served. A caller sees the union of tiers it is entitled to, and nothing more.

### Tenet 3: Verifiable Provenance for Every Static Claim
- Supplier-declared values (carbon footprint, recycled content shares, due diligence report, material composition) are accepted only as signed W3C Verifiable Credentials whose issuer DID resolves and whose signature validates at ingestion time.
- Unsigned or unverifiable claims are rejected, not stored as "pending trust".

### Tenet 4: Monotonic, Append-Only Lifecycle History
- Lifecycle status moves only forward: `ORIGINAL → REPURPOSED | REMANUFACTURED → WASTE → RECYCLED`. Backward transitions are prohibited.
- Dynamic values (SoH, cycle count, energy throughput) are appended as timestamped readings; earlier readings are never overwritten.

### Tenet 5: Resolver Availability Outlives the Operator
- The passport must remain resolvable if the issuing economic operator becomes insolvent or leaves the market. Every passport is replicated to an independent backup host and its identifier resolves through a registry decoupled from any single operator's infrastructure.

## 3. Scope Boundaries

### What We Are Building
- Passport issuance, unique identifier minting, and QR data carrier generation (GS1 Digital Link URIs, ISO/IEC 18004 QR).
- Role-based passport resolver serving Annex XIII attributes per access tier.
- Verifiable Credential ingestion and verification for supplier and conformity claims.
- BMS telemetry ingestion for SoH, cycle count, and remaining capacity updates.
- Lifecycle state machine with passport succession on repurposing and remanufacturing.

### What We Are NOT Building
- We do not produce corporate sustainability reports (CSRD / ESRS); this platform holds per-product lifecycle data only.
- We do not calculate product carbon footprints or perform due diligence audits; we store and verify the signed results.
- We do not act as a notified body, issue EU declarations of conformity, or replace the battery's on-board BMS.
