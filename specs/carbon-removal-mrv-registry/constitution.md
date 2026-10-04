# Constitution: Carbon Removal MRV & Credit Registry Platform

## 1. Purpose & Mission
The Carbon Removal MRV & Credit Registry Platform is a high-integrity system of record for durable and nature-based carbon dioxide removal (CDR) projects. It registers projects under approved methodologies (biochar, DACCS, enhanced rock weathering, BECCS), quantifies net removals from monitored data with conservative uncertainty treatment, orchestrates independent third-party validation and verification, and issues, transfers, retires, and cancels serialized removal units. Every unit must satisfy the EU Carbon Removals and Carbon Farming (CRCF) Regulation (EU) 2024/3012 quality criteria and the ICVCM Core Carbon Principles (CCPs), and must never be claimed twice.

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: Conservative Net Removal Quantification
- Units may only be issued against verified net removals, after deducting baseline removals, project lifecycle emissions, leakage, an uncertainty discount, and the buffer pool contribution:
  $$R_{\text{net}} = \left(R_{\text{gross}} - R_{\text{baseline}} - E_{\text{lifecycle}} - E_{\text{leakage}}\right) \times (1 - d_{\text{unc}}), \qquad R_{\text{issuable}} = \lfloor R_{\text{net}} \times (1 - b_{\text{risk}}) \rfloor$$
- If $R_{\text{net}} \le 0$ for a monitoring period, zero units are issued and the deficit is carried forward; negative balances are never netted against future periods silently.

### Tenet 2: Conservation of Serialized Units
- One unit equals exactly 1 tCO2e removed. Every unit belongs to exactly one serial range and exactly one state at any instant:
  $$\sum \text{Issued} = \sum \text{Active} + \sum \text{Retired} + \sum \text{Cancelled} + \sum \text{Buffer}$$
- Serial ranges are never re-used, re-numbered, or deleted. Transfers split ranges; they never mint or destroy units.

### Tenet 3: No Double Counting (Issuance, Use, Claim)
- A tonne may be issued once (no double issuance across registries), retired once (no double use), and claimed once (no double claiming between a buyer and a host country NDC).
- Units authorized for Paris Agreement Article 6 use MUST carry a corresponding adjustment status; units without one MUST be labelled as mitigation contributions and cannot be used toward another Party's NDC or CORSIA.

### Tenet 4: Independent Verification Before Issuance
- No issuance occurs without a positive verification statement from an accredited Validation & Verification Body (VVB) that has no conflict of interest with the project and has not verified the same project for more than the configured rotation limit.

### Tenet 5: Reversal Liability & Append-Only Registry Journal
- Every reversal (e.g., leakage from geological storage, biochar degradation beyond the permanence model) MUST be compensated by cancelling an equal tonnage from the buffer pool or the liable account. All state changes are written to an append-only, hash-chained journal.

## 3. Scope Boundaries

### What We Are Building
- Project registration, methodology versioning, and monitoring-period data ingestion (meters, lab assays, storage site reports).
- Net removal quantification engine with uncertainty, leakage, lifecycle (LCA) accounting, and buffer pool management.
- VVB validation/verification workflow and serialized unit issuance, transfer, retirement, and cancellation.
- Article 6 authorization tagging, cross-registry duplicate detection, and a public read-only registry API.

### What We Are NOT Building
- We do not produce corporate GHG inventories, Scope 1–3 accounting, or CSRD/ESRS disclosures (handled by the `csrd-esrs-sustainability` platform).
- We do not operate an exchange, order book, or price discovery venue, and we do not hold buyer funds.
