# Requirements Specification: Carbon Removal MRV & Credit Registry Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **Project Developer (ACT-DEV)**: Registers CDR projects, selects methodologies, submits monitoring reports, and receives issued units into a holding account.
- **Validation & Verification Body Auditor (ACT-VVB)**: Accredited third party (ISO 14065 / ISO 14064-3) that validates project design and verifies monitoring-period removals.
- **Registry Administrator (ACT-ADM)**: Approves methodologies, reviews issuance requests, manages the buffer pool, and executes reversal cancellations.
- **Account Holder / Buyer (ACT-BUY)**: Holds, transfers, and retires units on behalf of a named beneficiary for a voluntary or compliance claim.
- **Host Country Authority (ACT-HCA)**: Designated national authority that issues Article 6 letters of authorization and confirms corresponding adjustments.
- **MRV Data Feed Daemon (ACT-MRV)**: Ingests sensor telemetry, lab assay certificates, and storage operator injection reports via API and SFTP.

---

## 2. Functional Requirements

### 2.1 Project Registration & Methodologies (FR-PRJ)
- **FR-PRJ-01 (Methodology Catalog)**: The system MUST maintain versioned methodologies per pathway (biochar, DACCS, BECCS, enhanced rock weathering) with their quantification equations, default emission factors, minimum monitoring frequency, and CRCF activity type (permanent removal, carbon farming, carbon storage in products).
- **FR-PRJ-02 (Project Design Document Intake)**: The system MUST capture a Project Design Document (PDD) including geolocation (WGS84 polygon or storage site ID), crediting period start/end, baseline scenario, additionality demonstration, and storage permit reference (e.g., CO2 storage permit under Directive 2009/31/EC for geological storage).
- **FR-PRJ-03 (Additionality Test)**: The system MUST require a regulatory surplus test and a financial or barrier test; projects mandated by law or financially viable without carbon revenue MUST be rejected with reason code `NOT_ADDITIONAL`.
- **FR-PRJ-04 (Sustainability Safeguards)**: The system MUST record do-no-significant-harm evidence and sustainability co-benefit claims against CRCF sustainability objectives (biodiversity, water, circular economy, pollution) before validation can complete.

### 2.2 Net Removal Quantification (FR-QNT)
- **FR-QNT-01 (Net Removal Calculation)**: For each monitoring period, the system MUST compute:
  $$R_{\text{net}} = R_{\text{gross}} - R_{\text{baseline}} - E_{\text{lifecycle}} - E_{\text{leakage}}$$
  where $E_{\text{lifecycle}}$ covers cradle-to-grave emissions (energy, feedstock transport, capture, compression, transport, injection, spreading) per ISO 14040/14044.
- **FR-QNT-02 (Biochar Permanence Factor)**: For biochar, the system MUST derive the persistent fraction from lab-measured H/C$_{\text{org}}$ ratio ($\le 0.7$ required) and soil temperature using the methodology's decay model, applied over a minimum 100-year horizon.
- **FR-QNT-03 (Uncertainty Discount)**: The system MUST compute the 90% confidence interval of $R_{\text{net}}$ via Monte Carlo ($N \ge 10{,}000$ draws) and apply a discount $d_{\text{unc}}$: 0% if relative half-width $\le 10\%$, otherwise equal to the half-width percentage minus 10%; if the discount would exceed 50% (half-width $> 60\%$) the period MUST be rejected.
- **FR-QNT-04 (Deficit Carry-Forward)**: If $R_{\text{net}} \le 0$, the system MUST issue zero units and carry the negative balance forward to be deducted from subsequent periods.

### 2.3 Buffer Pool & Reversal Management (FR-BUF)
- **FR-BUF-01 (Risk-Based Buffer Contribution)**: The system MUST withhold $b_{\text{risk}}$ of issuable units into a pooled buffer account, scored per pathway: geological DACCS/BECCS 2–5%, biochar 10–15%, enhanced rock weathering 15–20%, using a non-permanence risk tool.
- **FR-BUF-02 (Reversal Cancellation)**: On a reported reversal, the system MUST cancel buffer units equal to the reversed tonnage within 5 business days; intentional reversals MUST additionally debit the developer's account or trigger a replenishment obligation.
- **FR-BUF-03 (Buffer Release)**: Buffer units MUST only be released or reallocated by ACT-ADM dual authorization after the methodology's monitoring obligation (e.g., post-closure liability transfer for geological storage) is satisfied.

### 2.4 Validation & Verification Workflow (FR-VER)
- **FR-VER-01 (VVB Assignment & Conflict Check)**: The system MUST verify VVB accreditation scope and expiry, block assignment where the VVB performed consultancy for the project, and enforce rotation after 6 consecutive verifications.
- **FR-VER-02 (Findings Lifecycle)**: VVBs MUST raise Corrective Action Requests (CAR), Clarification Requests (CL), and Forward Action Requests (FAR); issuance is blocked while any CAR or CL is open.
- **FR-VER-03 (Signed Verification Statement)**: The verification statement MUST state a reasonable or limited assurance level, the verified tonnage, and a materiality threshold ($\le 5\%$) and be signed with an X.509 qualified signature.

### 2.5 Serialized Unit Lifecycle (FR-UNT)
- **FR-UNT-01 (Serialized Issuance)**: The system MUST issue one unit per whole tCO2e with globally unique serial ranges formatted `<Registry>-<ProjectID>-<Vintage>-<Pathway>-<Start>-<End>` (e.g., `CRX-00412-2026-DAC-000000001-000025000`).
- **FR-UNT-02 (Transfer with Range Splitting)**: Partial transfers MUST split the source range into contiguous sub-ranges atomically; the sum of quantities before and after MUST be equal.
- **FR-UNT-03 (Retirement with Beneficiary)**: Retirement MUST record beneficiary legal name, LEI or national ID, claim purpose (voluntary, CORSIA, NDC), and reporting year, and produce a public, immutable retirement certificate.
- **FR-UNT-04 (Cancellation)**: The system MUST support administrative cancellation (reversal compensation, error correction, transfer to another registry) with reason code; cancelled units can never be reactivated.

### 2.6 Double-Counting Prevention & Article 6 (FR-DCP)
- **FR-DCP-01 (Cross-Registry Duplicate Detection)**: Before issuance, the system MUST match project geolocation, storage site ID, and feedstock batch IDs against the internal registry and external registry feeds; overlaps $> 0\%$ block issuance.
- **FR-DCP-02 (Corresponding Adjustment Flag)**: Each unit MUST carry `authorization_status` (`UNAUTHORIZED_MITIGATION_CONTRIBUTION`, `AUTHORIZED_NDC`, `AUTHORIZED_OIMP`) and `ca_status` (`NOT_REQUIRED`, `PENDING`, `APPLIED`) linked to a host-country letter of authorization.
- **FR-DCP-03 (Use Restriction Enforcement)**: Retirement for NDC or CORSIA purposes MUST be rejected unless `ca_status = APPLIED` or an equivalent host-country confirmation exists.

### 2.7 Public Registry API (FR-API)
- **FR-API-01 (Read-Only Public API)**: The system MUST expose projects, methodologies, verification statements, issuances, retirements, and buffer balances via REST/JSON with cursor pagination and CC-BY open data licensing.
- **FR-API-02 (Serial Lookup)**: Any single serial number MUST resolve to its range, project, vintage, current state, and (if retired) beneficiary and claim purpose.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance & Scalability (NFR-PERF)
- **NFR-PERF-01 (Ledger Throughput)**: The unit ledger MUST sustain 2,000 transfer/retirement transactions per second with p99 commit latency $\le 150\text{ms}$.
- **NFR-PERF-02 (Quantification Runtime)**: A 10,000-draw Monte Carlo quantification for a 50,000-sample monitoring period MUST complete in $\le 60\text{ seconds}$.
- **NFR-PERF-03 (Public API Latency)**: Serial lookup MUST respond in $\le 100\text{ms}$ p95 at 5,000 requests per second.

### 3.2 Integrity & Security (NFR-SEC)
- **NFR-SEC-01 (Hash-Chained Journal)**: Every ledger event MUST include the SHA-256 hash of the prior event; daily Merkle roots MUST be anchored to an external timestamping authority (RFC 3161).
- **NFR-SEC-02 (Dual Authorization)**: Issuance, buffer release, and administrative cancellation MUST require two distinct ACT-ADM approvals with FIDO2 hardware-key MFA.

### 3.3 Reliability (NFR-REL)
- **NFR-REL-01 (Idempotent Transactions)**: All mutating API calls MUST accept an idempotency key; replays MUST return the original result without duplicating units.
- **NFR-REL-02 (Availability & Recovery)**: Registry ledger availability MUST be $\ge 99.95\%$ with RPO = 0 (synchronous replication) and RTO $\le 15\text{ minutes}$.

---

## 4. Removal Unit Lifecycle

```
[Project Developer: PDD + Methodology]
                |
                v
     [VVB Validation (Additionality / Safeguards)] --(CAR open)--> [Revise PDD]
                |
                v
     [Registered Project] <-------- [MRV Feeds: Sensors / Lab Assays / Injection Reports]
                |
                v
     [Monitoring Report: R_gross - Baseline - LCA - Leakage]
                |
                v
     [Uncertainty Discount (Monte Carlo 90% CI)] --(R_net <= 0)--> [Deficit Carry-Forward]
                |
                v
     [VVB Verification Statement] --> [Duplicate & Article 6 Checks]
                |
                v
     [Dual-Approval Issuance: Serial Range Minted]
                |
       +--------+-------------------+
       |                            |
       v                            v
 [Buffer Pool (b_risk %)]   [Developer Holding Account]
       |                            |
 (Reversal Event)          +--------+--------+
       |                   |                 |
       v                   v                 v
 [Buffer Cancellation] [Transfer (Split)] [Retire w/ Beneficiary]
                                             |
                                             v
                               [Public Retirement Certificate]
```
