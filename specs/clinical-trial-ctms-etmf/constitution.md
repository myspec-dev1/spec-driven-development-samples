# Constitution: Clinical Trial Management System (CTMS) & 21 CFR Part 11 Compliant eTMF

## 1. Purpose & Mission
The Clinical Trial Management System (CTMS) and Electronic Trial Master File (eTMF) platform provides clinical research sponsors, Contract Research Organizations (CROs), and investigative trial sites with a compliant, auditable, cloud-native operational system for Phase I-IV and decentralized clinical trials (DCT). The platform enforces strict regulatory compliance under FDA 21 CFR Part 11, EU Annex 11, and ICH GCP E6(R2/R3).

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: Strict FDA 21 CFR Part 11 & EU Annex 11 Electronic Signature Integrity
- Any document signing, approval, protocol deviation clearance, or eCRF verification must enforce dual-credential authorization (password/passcode + time-based OTP or biometric).
- The visual signature manifestation must explicitly embed: printed name of signer, date/time in UTC with timezone offset, and the unambiguous regulatory meaning of the signature (`Authored`, `Reviewed`, `Approved`, or `Verified`).
- Signatures are cryptographically bonded to the specific SHA-256 hash of the document snapshot; modifying even 1 byte invalidates the signature verification state.

### Tenet 2: Immutable ALCOA+ Data Integrity & Computer-Generated Audit Trails
- All system records must satisfy ALCOA+ tenets: Attributable, Legible, Contemporaneous, Original, Accurate, Complete, Consistent, Enduring, and Available.
- The audit trail must be computer-generated, independent of human intervention, recording every create, read, update, signature, download, and archive action with timestamp and user ID.
- No user, including the root/database administrator, can alter or truncate audit trail entries.

### Tenet 3: Strict Blinding & Randomization Firewalls
- The platform must mathematically isolate blinded investigators, monitors, and participants from unblinded study team members (e.g. investigational product preparation pharmacists).
- Under no operational failure or query shall unblinded kit identifiers, drug active/placebo assignments, or kit sequence codes be exposed to blinded user roles prior to formal database lock and unblinding approval.

### Tenet 4: Zero-Tolerance Serious Adverse Event (SAE) 24-Hour Escalation
- When an Adverse Event is flagged as "Serious" (death, life-threatening, hospitalization, disability, or congenital anomaly), the platform enters a mission-critical expedited reporting state.
- Automatic notifications to the Principal Investigator, Sponsor Medical Monitor, and Pharmacovigilance (PV) safety team must trigger immediately, with hard escalation timers tracking the FDA 7-day and 15-day expedited reporting deadlines.

### Tenet 5: Standard DIA TMF Reference Model v3.2 Taxonomy
- All eTMF document structures, metadata zones, sections, and artifact numbers must conform strictly to the standard Drug Information Association (DIA) TMF Reference Model (Zones 01 through 11).

## 3. Scope Boundaries

### What We Are Building
- Protocol builder, subject visit scheduling, site feasibility, and investigator payment milestone tracking.
- Electronic Trial Master File (eTMF) document ingestion, metadata classification, QC review, and export.
- Regulatory compliance engine (21 CFR Part 11 signatures, computer-generated WORM audit trails).
- Pharmacovigilance SAE expedited incident escalation workflow.
- CDISC ODM (Operational Data Model) / FHIR clinical data interoperability connectors.

### What We Are NOT Building
- We do not manufacture physical investigational drug kits or manage cold-chain temperature loggers directly (we interface via IoT webhooks and IRT/RTSM integrations).
- We do not provide medical advice or automate clinical diagnosis decisions.
