# Constitution: DSCSA Pharmaceutical Serialization & Traceability Platform

## 1. Purpose & Mission
The DSCSA Pharmaceutical Serialization & Traceability Platform is a regulated, multi-tenant system of record that lets manufacturers, repackagers, wholesale distributors, and dispensers meet the US Drug Supply Chain Security Act (DSCSA) enhanced drug distribution security requirements: interoperable, electronic, package-level tracing of prescription drugs. It captures GS1 EPCIS 2.0 events for every serialized unit, exchanges Transaction Information (TI) and Transaction Statements (TS) only with Authorized Trading Partners, verifies product identifiers for saleable returns, and drives suspect and illegitimate product investigations to closure. The EU Falsified Medicines Directive (FMD) is supported as a secondary market through EMVS upload and decommissioning.

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: One Serial, One Life
- Every product identifier (GTIN + serial number, with lot and expiry) is commissioned exactly once and follows a single, ordered event history. A serial can never be active in two places at the same time.
- Once a serial is decommissioned (dispensed, destroyed, stolen, or sampled), any later shipping or receiving event for it is rejected and raises a suspect-product alert.

### Tenet 2: Aggregation Conservation
- For every parent container (case or pallet SSCC), the set of saleable units reachable through the hierarchy must exactly equal the units physically packed, at every point in time:
  $$\forall p:\ \text{Units}(p) = \bigcup_{c \in \text{children}(p)} \text{Units}(c), \qquad \text{Units}(c_i) \cap \text{Units}(c_j) = \emptyset \ \ (i \neq j)$$
- Inference ("ship the pallet, trust the contents") is allowed only while the hierarchy is intact. Any unpack, damage, or mismatch breaks inference and forces unit-level events.

### Tenet 3: Append-Only, Six-Year Evidence
- EPCIS events, TI/TS documents, verification requests, and investigation records are immutable and kept for at least 6 years from the transaction date.
- Corrections are made with EPCIS error declarations or new events that reference the original, never by editing or deleting history.

### Tenet 4: Trade Only With Authorized Trading Partners
- No TI/TS is sent, no ownership-change event is accepted, and no saleable return is restocked unless the counterparty's state license or FDA registration is verified and current.

### Tenet 5: Regulatory Clocks Are Hard Deadlines
- Suspect and illegitimate product workflows run on enforced timers: FDA Form 3911 notification within 24 hours of determining a product is illegitimate, and tracing-request responses within 1 business day (never more than 48 hours).

## 3. Scope Boundaries

### What We Are Building
- Serial number management, GS1 DataMatrix / SGTIN encoding, and EPCIS 2.0 event repository with CBV vocabulary.
- Aggregation hierarchy service (unit → case → pallet SSCC) with inference and integrity checks.
- TI/TS exchange, Authorized Trading Partner verification, and VRS requester/responder for saleable returns.
- Suspect/illegitimate product investigation workbench, Form 3911 workflow, and fast tracing-request responder.
- EU FMD connector for EMVS (EU Hub / national systems) upload, verification, and decommissioning.

### What We Are NOT Building
- We do not run warehouse operations (picking, putaway, slotting, labor); we consume events from WMS and packaging lines.
- We do not print labels or control packaging-line hardware (Level 2/3 line systems remain the source of commissioning events).
- We are not a pharmacy dispensing or claims system, and we do not replace the FDA or EMVO portals themselves.
