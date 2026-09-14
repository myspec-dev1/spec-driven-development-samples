# Requirements Specification: Clinical Trial Management System (CTMS) & 21 CFR Part 11 Compliant eTMF

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **Clinical Research Associate / Monitor (ACT-CRA)**: Conducts site initiation, interim monitoring, and close-out visits; logs Trip Reports and verifies eTMF binders.
- **Principal Investigator / Site Coordinator (ACT-INV)**: Recruits patients, schedules protocol visits, records adverse events, and signs eCRFs.
- **Trial Master File Manager / Document Specialist (ACT-TMF)**: Performs Quality Control (QC) checks on uploaded trial documents, inspects completeness, and runs audit inspections.
- **Safety / Pharmacovigilance Officer (ACT-SAF)**: Evaluates Serious Adverse Events (SAEs), adjudicates causality, and files MedWatch 3500A / CIOMS I forms.
- **Study Director / Clinical Project Manager (ACT-CPM)**: Tracks site recruitment milestones, manages subject visit funnels, and approves site investigator budget disbursements.
- **Auditor / Regulatory Inspector (ACT-AUD)**: External FDA, EMA, or IRB inspector conducting read-only inspection audits.

---

## 2. Functional Requirements

### 2.1 Study Protocol, Site Management & Subject Funnel (FR-CTM)
- **FR-CTM-01 (Study Protocol & Visit Matrix Configuration)**: The system MUST allow study designers to configure the protocol visit schedule (Screening, Baseline, Visit 1..N, Unscheduled, End of Study) with allowable visit window tolerances (e.g. Day $14 \pm 2$ days).
- **FR-CTM-02 (Site Feasibility & Activation Checklist)**: The system MUST track site regulatory greenlight milestones (IRB/IEC approval, executed Clinical Trial Agreement, Form FDA 1572, CVs, Financial Disclosures) and prevent subject enrollment until all mandatory site documents are marked `APPROVED`.
- **FR-CTM-03 (Subject Recruitment & Visit Tracking)**: The system MUST track subject progression through state transitions: `Candidate -> Consented -> Screened -> Randomized -> Active -> Completed / Discontinued / Lost to Follow-Up`.
- **FR-CTM-04 (Subject Visit Deviation Detection)**: If an in-person or telehealth visit is scheduled outside the protocol allowable window ($>\text{Upper Limit}$ or $<\text{Lower Limit}$), the system MUST automatically generate a `Minor Protocol Deviation` ticket and require CRA review.

### 2.2 21 CFR Part 11 Electronic Records & Signatures (FR-SIG)
- **FR-SIG-01 (Dual-Credential Digital Signing)**: Electronic signatures MUST require explicit user re-authentication with two distinct credentials (e.g. user password + time-based TOTP code or passkey).
- **FR-SIG-02 (Immutable Signature Bond & Visual Manifestation)**: The system MUST calculate the SHA-256 hash of the target PDF/document before signing. The resulting signed artifact MUST embed a visual signature page detailing:
  - Full legal name and role of signer
  - UTC timestamp with ISO 8601 offset
  - Regulatory intent code: `I have reviewed and approve this clinical document`
  - Cryptographic verification signature ID
- **FR-SIG-03 (Session Timeout & Account Security)**: The system MUST automatically terminate inactive user sessions after 15 minutes of inactivity. After 3 consecutive failed login attempts, user accounts MUST be locked for 30 minutes, logging a security event.

### 2.3 Electronic Trial Master File (eTMF) & DIA Taxonomy (FR-TMF)
- **FR-TMF-01 (Standard DIA TMF Reference Model v3.2)**: The eTMF hierarchy MUST enforce the 11 DIA standard zones:
  - Zone 01: Trial Management
  - Zone 02: Central Trial Documents
  - Zone 03: Regulatory
  - Zone 04: IRB / IEC
  - Zone 05: Site Management
  - Zone 06: IP & Trial Supplies
  - Zone 07: Safety Reporting
  - Zone 08: Central & Local Lab
  - Zone 09: Third Parties
  - Zone 10: Data Management
  - Zone 11: Statistics
- **FR-TMF-02 (Document Lifecycle State Machine)**: Documents MUST follow strict states: `Draft -> Uploaded -> QC Pending -> QC Passed / QC Rejected -> Final Approved -> Superseded / Archived`.
- **FR-TMF-03 (TMF Completeness & Milestone Quality Index)**: The system MUST calculate the Real-time TMF Completeness score:
  $$\text{Completeness} = \frac{\sum \text{Approved Mandatory Artifacts}}{\sum \text{Expected Protocol Artifacts}} \times 100\%$$
  flagging missing expected documents based on current study milestones.
- **FR-TMF-04 (Inspection Room Mode for Regulatory Auditors)**: The system MUST support an "Auditor View" granting time-limited (e.g. 72-hour), watermarked, read-only access to specific zones with an exportable chronological audit log.

### 2.4 Pharmacovigilance & 24-Hour SAE Escalation (FR-SAF)
- **FR-SAF-01 (SAE Immediate Entry & Flagging)**: When an Adverse Event form is checked with any seriousness criteria (Death, Inpatient Hospitalization, Congenital Anomaly, Persistent Disability, Life-Threatening), the system MUST trigger the SAE Escalation Workflow within 60 seconds.
- **FR-SAF-02 (24-Hour Notification Clock)**: The system MUST start a countdown timer requiring initial Safety Review and medical monitor acknowledgment within 24 hours of site entry. If unacknowledged at 18 hours, an automated SMS/Phone escalation is sent to the secondary medical director.
- **FR-SAF-03 (Automated MedWatch / CIOMS Export)**: The safety module MUST generate pre-populated FDA Form 3500A (MedWatch) and CIOMS I PDF forms for expedited regulatory submission.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Regulatory Compliance & Validation (NFR-REG)
- **NFR-REG-01 (GAMP 5 Validation Ready)**: System software MUST provide complete Computerized Systems Validation (CSV) documentation packages: IQ (Installation Qualification), OQ (Operational Qualification), and PQ (Performance Qualification).
- **NFR-REG-02 (Computer-Generated WORM Audit Trail)**: Audit logs MUST be written to immutable append-only object storage. No user, database operator, or API token can update or delete audit records.

### 3.2 Security, Privacy & Data Protection (NFR-SEC)
- **NFR-SEC-01 (HIPAA & GDPR De-Identification)**: All subject medical records MUST be pseudonymous using Subject Screening IDs (e.g. `SITE01-SUB042`). No real names, addresses, or government IDs can appear in eTMF documents without cryptographic de-identification.
- **NFR-SEC-02 (Data At Rest & Transit Encryption)**: AES-256 encryption at rest; TLS 1.3 enforced for all web, mobile, and API data transfers.

### 3.3 Performance & Document Processing (NFR-PERF)
- **NFR-PERF-01 (Document Rendering & Watermarking)**: PDF rendition, OCR indexing, and compliance watermarking of uploaded files ($< 50\text{MB}$) MUST complete in $\le 5\text{ seconds}$.
- **NFR-PERF-02 (Audit Trail Query Latency)**: Historical audit search across 10,000,000 log lines MUST return filtered results in $\le 1.5\text{ seconds}$.

---

## 4. SAE Escalation State Machine

```
[Site Enters Adverse Event] -> [Seriousness Check: YES]
                                     |
                                     v
                       [Start 24-Hour Regulatory Clock]
                                     |
                   +-----------------+-----------------+
                   |                                   |
         (T <= 18 Hours)                       (T > 18 Hours: No Ack)
                   |                                   |
         [Medical Monitor Reviews]             [High-Priority SMS Escalation]
                   |                                   |
         [Adjudicate Causality]                [Backup Safety Director Alert]
                   |                                   |
         [Generate CIOMS / MedWatch]                   v
                   |                         [Medical Monitor Acknowledged]
                   v
         [File FDA/EMA Expedited Report]
```
