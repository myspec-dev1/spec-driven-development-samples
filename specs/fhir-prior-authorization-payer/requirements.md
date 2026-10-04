# Requirements Specification: FHIR Payer Interoperability & Electronic Prior Authorization Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **Plan Member / Patient (ACT-MBR)**: Accesses claims, clinical data, and prior authorization status through a third-party app; grants or withholds Payer-to-Payer consent.
- **Ordering Provider / EHR (ACT-PRV)**: Places orders in a certified EHR that invokes CRD hooks, launches DTR, and submits PAS requests.
- **Utilization Management Nurse Reviewer (ACT-UMR)**: Reviews pended requests, requests additional documentation, and records approve/deny/modify decisions with specific reasons.
- **Medical Director (ACT-MDR)**: Physician reviewer who must sign every adverse (denial) determination based on medical necessity.
- **Other Health Plan (ACT-OPP)**: Previous or concurrent payer requesting or supplying member data via the Payer-to-Payer API.
- **Compliance & Reporting Officer (ACT-CMP)**: Publishes annual prior authorization metrics and responds to CMS / state regulator audits.
- **Clearinghouse / X12 Gateway (ACT-CHG)**: Exchanges HIPAA X12 278 request/response transactions with the payer's legacy UM system.

---

## 2. Functional Requirements

### 2.1 Patient Access API (FR-PAT)
- **FR-PAT-01 (Member Data Scope)**: The system MUST expose adjudicated claims and encounters (CARIN Blue Button), USCDI clinical data (US Core), and plan formulary data via FHIR R4 to member-authorized apps using SMART App Launch.
- **FR-PAT-02 (Prior Authorization Status)**: The system MUST expose each non-drug prior authorization (status, decision date, approved quantity/date range, and denial reason) no later than 1 business day after the payer receives the request or the status changes.
- **FR-PAT-03 (Retention Window)**: Prior authorization records MUST remain available through the Patient Access API for as long as the authorization is active and for at least 1 year after the last status change.

### 2.2 Provider Access API (FR-PRV)
- **FR-PRV-01 (Attribution-Gated Access)**: The system MUST release member data only to in-network providers with a verified treatment relationship, resolved through a maintained attribution list (Da Vinci ATR `Group` resources).
- **FR-PRV-02 (Bulk Export)**: The system MUST support FHIR Bulk Data `Group/[id]/$export` returning NDJSON for attributed members, with data available within 1 business day of the provider request.
- **FR-PRV-03 (Member Opt-Out)**: Members who opt out MUST be excluded from all Provider Access responses within $\le 1$ business day of the opt-out being recorded.

### 2.3 Payer-to-Payer Exchange (FR-P2P)
- **FR-P2P-01 (Explicit Opt-In)**: The system MUST present the opt-in request to new members no later than 1 week after the start of coverage and record the decision (consent version, timestamp, channel); no data MUST be requested from a previous payer without a recorded opt-in.
- **FR-P2P-02 (Five-Year Lookback)**: The system MUST request and supply claims/encounter data (excluding provider remittances and member cost-sharing), USCDI data, and active/pending prior authorizations with dates of service within the prior 5 years.
- **FR-P2P-03 (Concurrent Coverage)**: For members with concurrent coverage, the system MUST exchange data with each concurrent payer at least quarterly.

### 2.4 Coverage Requirements Discovery & Documentation (FR-CRD)
- **FR-CRD-01 (CDS Hooks Services)**: The system MUST implement CRD CDS Services for `order-select`, `order-sign`, and `appointment-book` hooks, returning cards indicating whether prior authorization is required within $\le 1\text{ second}$ (p95).
- **FR-CRD-02 (Coverage Information Extension)**: Responses MUST attach the CRD `coverage-information` extension to the draft order (covered, PA needed, documentation needed) so it travels into DTR and PAS.
- **FR-CRD-03 (DTR SMART Launch)**: When documentation is required, the system MUST serve the applicable `Questionnaire` and CQL libraries via `Questionnaire/$questionnaire-package`, pre-populating answers from the EHR and launching through SMART on FHIR.
- **FR-CRD-04 (Adaptive Questionnaires)**: The system MUST support adaptive forms via `$next-question`, and the completed `QuestionnaireResponse` MUST be bundled into the PAS request.

### 2.5 Prior Authorization Submission & Decisioning (FR-PAS)
- **FR-PAS-01 (PAS Submit)**: The system MUST accept a PAS `Bundle` via `Claim/$submit` (use = `preauthorization`) and return a synchronous `ClaimResponse` with an approved, denied, modified, or pended outcome.
- **FR-PAS-02 (X12 278 Translation)**: The system MUST translate PAS Bundles to and from X12 278 (005010X217) losslessly for payers whose UM system or HIPAA compliance posture requires the X12 standard.
- **FR-PAS-03 (Decision Timeframes)**: Final decisions MUST be issued within $\le 72\text{ hours}$ for expedited and $\le 7\text{ calendar days}$ for standard requests, measured from receipt; breaches MUST page the UM supervisor at 80% of the window.
- **FR-PAS-04 (Specific Denial Reason)**: Every denied or modified item MUST carry a coded reason and a free-text explanation referencing the applicable medical policy; a denial MUST NOT be finalized without one.
- **FR-PAS-05 (Status Inquiry & Subscription)**: The system MUST support `Claim/$inquire` and FHIR Subscriptions so EHRs receive pended-to-final updates without polling.

### 2.6 Prior Authorization Metrics Reporting (FR-RPT)
- **FR-RPT-01 (Public Metrics)**: The system MUST compute, per calendar year and excluding drugs, the percentage of standard and expedited requests approved and denied, approved after appeal, and approved after an extended timeframe, plus average and median decision time:
  $$\bar{t}_{\text{std}} = \frac{1}{N_{\text{std}}} \sum_{i=1}^{N_{\text{std}}} \left( t_{\text{decision},i} - t_{\text{received},i} \right)$$
- **FR-RPT-02 (Annual Publication)**: Metrics MUST be exported as a machine-readable dataset and an accessible HTML page for posting on the payer's public website by 31 March each year.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance & Scalability (NFR-PERF)
- **NFR-PERF-01 (CRD Latency)**: CDS Hooks responses MUST complete in $\le 1\text{ s}$ p95 and $\le 2\text{ s}$ p99 at 500 hook calls/second.
- **NFR-PERF-02 (Bulk Export Throughput)**: A Provider Access `$export` for a 50,000-member attribution group MUST complete in $\le 30\text{ minutes}$.

### 3.2 Security & Privacy (NFR-SEC)
- **NFR-SEC-01 (SMART on FHIR Authorization)**: All APIs MUST use OAuth 2.0 with SMART App Launch scopes (v2 granular scopes) and OpenID Connect identity; system-to-system access MUST use SMART Backend Services with `private_key_jwt` assertions.
- **NFR-SEC-02 (HIPAA Safeguards)**: PHI MUST be encrypted with TLS 1.2+ in transit and AES-256 at rest; every PHI disclosure MUST be written to an append-only audit log (FHIR `AuditEvent`) retained for 6 years.

### 3.3 Reliability & Availability (NFR-REL)
- **NFR-REL-01 (API Availability)**: Prior Authorization and Patient Access APIs MUST maintain $\ge 99.9\%$ monthly availability.
- **NFR-REL-02 (Idempotent Submission)**: Resubmitting an identical PAS Bundle (same `Bundle.identifier`) MUST return the original `ClaimResponse` without creating a duplicate request or restarting the decision clock.

---

## 4. Electronic Prior Authorization Pipeline (CRD → DTR → PAS)

```
[Provider EHR: order-select / order-sign]
                 |
                 v
     [CRD CDS Service (CDS Hooks)] ----> PA not required ----> [Card: "No PA needed"] --> Order proceeds
                 |
          PA / docs required
                 v
  [DTR SMART App: $questionnaire-package + CQL prepopulation]
                 |
                 v
     [QuestionnaireResponse bundled into PAS Bundle]
                 |
                 v
       [PAS Claim/$submit] ----------------> [X12 278 Translator] ---> [Legacy UM System]
                 |                                                            |
                 v                                                            |
   [Decision Clock Started: 72h expedited | 7d standard] <--------------------+
                 |
     +-----------+-------------+------------------+
     |                         |                  |
 [Approved (A1)]        [Pended (A4)]       [Denied (A3) + specific reason]
     |                         |                  |
     v                         v                  v
[ClaimResponse to EHR]  [Request docs ->   [Medical Director sign-off]
     |                   Subscription]            |
     +-------------+-----------+------------------+
                   |
                   v
 [Patient Access API (<= 1 business day)]  +  [Metrics Journal -> Annual Public Report]
```
