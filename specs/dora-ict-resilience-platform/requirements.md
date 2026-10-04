# Requirements Specification: DORA ICT Risk & Operational Resilience Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **ICT Risk Manager (ACT-IRM)**: Maintains the ICT asset & risk inventory, runs risk assessments, and owns the ICT risk management framework review.
- **Incident Manager (ACT-INC)**: Triages ICT-related incidents, confirms major-incident classification, and signs off supervisory reports.
- **Third-Party Risk Officer (ACT-TPR)**: Maintains contractual arrangements in the Register of Information and owns exit strategies and concentration assessments.
- **Resilience Testing Lead (ACT-TST)**: Plans annual resilience tests and coordinates TLPT engagements with threat-intelligence and red-team providers.
- **Management Body Member (ACT-MGB)**: Approves the ICT risk framework, digital operational resilience strategy, and TLPT attestations; bears ultimate accountability.
- **Competent Authority Gateway (ACT-NCA)**: External supervisor portal (e.g. BaFin, ACPR, DNB, CBI) receiving incident reports and register submissions.
- **ITSM / SIEM Integration Daemon (ACT-ING)**: Pushes incident tickets and alerts from ServiceNow, Jira Service Management, Splunk, or Microsoft Sentinel.

---

## 2. Functional Requirements

### 2.1 ICT Asset & Risk Inventory (FR-AST)
- **FR-AST-01 (Asset & Function Registry)**: The system MUST record ICT assets (applications, infrastructure, data stores, network segments) with owner, location, and classification, and map each asset to the business functions it supports, flagging critical or important functions (CIFs).
- **FR-AST-02 (Dependency Graph)**: The system MUST maintain a directed dependency graph (function → asset → ICT service → provider, including subcontractor chains) and compute the full transitive dependency set of any CIF in $\le 500\text{ms}$ for graphs of 100,000 nodes.
- **FR-AST-03 (Risk Scoring)**: The system MUST score inherent and residual ICT risk per asset as $R = L \times I$ on a 5×5 scale and require a documented treatment decision for any residual $R \ge 15$.

### 2.2 Incident Classification (FR-INC)
- **FR-INC-01 (Incident Intake)**: The system MUST ingest incidents via REST and webhooks from ITSM/SIEM tools, recording `detected_at` and `aware_at` as immutable UTC timestamps.
- **FR-INC-02 (RTS Major Classification)**: The system MUST classify an incident as major when it affects critical services AND either (a) involves malicious unauthorised access to network and information systems that may lead to data losses, or (b) meets the thresholds of at least two other criteria, e.g.:
  - Clients/counterparts affected $> 10\%$ or $> 100{,}000$; transactions affected $> 10\%$ of daily average or value $> €15\text{M}$.
  - Duration $> 24\text{h}$ or critical-service downtime $> 2\text{h}$; impact in $\ge 2$ Member States; economic impact $\ge €100{,}000$; any material data loss; reputational impact.
- **FR-INC-03 (Recurring Incidents)**: The system MUST aggregate non-major incidents sharing the same apparent root cause occurring at least twice within 6 months and re-evaluate them collectively as a single recurring major incident.
- **FR-INC-04 (Significant Cyber Threats)**: The system MUST allow voluntary notification of significant cyber threats using the authority's threat notification template.

### 2.3 Supervisory Incident Reporting (FR-RPT)
- **FR-RPT-01 (Initial Notification)**: The system MUST generate the initial notification within 4 hours of classification and no later than 24 hours from awareness of the incident.
- **FR-RPT-02 (Intermediate Report)**: The system MUST produce the intermediate report within 72 hours of the initial notification, and an updated intermediate report whenever regular activities are recovered or the status changes significantly.
- **FR-RPT-03 (Final Report)**: The system MUST produce the final report within 1 month of the latest intermediate report, including root cause, total economic impact, and remediation actions.
- **FR-RPT-04 (Client Notification)**: Where a major incident affects clients' financial interests, the system MUST draft a client communication without undue delay, listing mitigation measures taken.

### 2.4 Register of Information (FR-ROI)
- **FR-ROI-01 (ITS Templates)**: The system MUST maintain all Register of Information templates (`B_01.01` to `B_07.01`, plus `B_99.01`) as defined by the ITS on the Register of Information, at entity, sub-consolidated, and consolidated level.
- **FR-ROI-02 (Identifier Validation)**: The system MUST validate LEIs (ISO 17442, 20 characters, ISO 7064 MOD 97-10 checksum) against GLEIF and accept EUID where a provider has no LEI.
- **FR-ROI-03 (xBRL-CSV Export)**: The system MUST export the register as an xBRL-CSV report package (zip with `META-INF/reportPackage.json`, `reports/*.csv`, `parameters.csv`) and pass all ESA validation rules before submission.

### 2.5 Concentration Risk (FR-CON)
- **FR-CON-01 (Provider Concentration Index)**: The system MUST compute a Herfindahl-Hirschman Index of CIF dependency per ICT service type:
  $$\text{HHI} = \sum_{i=1}^{n} s_i^2 \times 10{,}000, \quad s_i = \frac{\text{CIFs served by provider } i}{\text{total CIFs}}$$
  and flag $\text{HHI} > 2{,}500$ as high concentration.
- **FR-CON-02 (Substitutability & Hidden Concentration)**: The system MUST surface providers reached through subcontracting chains and flag CIFs supported by non-substitutable providers.

### 2.6 Resilience Testing & TLPT (FR-TST)
- **FR-TST-01 (Annual Testing Programme)**: The system MUST track that all ICT systems supporting CIFs are tested at least yearly (vulnerability scans, scenario tests, failover, performance) with findings and remediation owners.
- **FR-TST-02 (TLPT Lifecycle)**: The system MUST track TLPT under the TIBER-EU phases (Preparation, Testing: Threat Intelligence + Red Team, Closure: Replay/Purple Team + Remediation) on a 3-year cycle, with a minimum active red-team phase of 12 weeks, and store the authority attestation.

### 2.7 Exit Strategies (FR-EXT)
- **FR-EXT-01 (Exit Plan Registry)**: The system MUST require a documented, owned exit plan for every contract supporting a CIF, including alternative providers, transition timeline, and data portability approach, re-tested at least yearly.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance (NFR-PERF)
- **NFR-PERF-01 (Classification Latency)**: Major-incident classification MUST complete in $\le 200\text{ms}$ p99 after intake.
- **NFR-PERF-02 (Register Export)**: xBRL-CSV export and validation for 50,000 contractual arrangements MUST complete in $\le 2\text{ minutes}$.

### 3.2 Security & Auditability (NFR-SEC)
- **NFR-SEC-01 (Tamper Evidence)**: Incident timelines and submitted reports MUST be hash-chained (SHA-256) and stored in WORM storage for at least 5 years.
- **NFR-SEC-02 (Access Control)**: Report submission and register release MUST require four-eyes approval with SSO + phishing-resistant MFA (FIDO2/WebAuthn).

### 3.3 Reliability (NFR-REL)
- **NFR-REL-01 (Deadline Watchdog Availability)**: The deadline watchdog MUST achieve 99.95% availability with RPO $= 0$ and RTO $\le 15\text{ minutes}$, deployed active-active across 2 EU regions.

---

## 4. Major Incident Classification & Reporting Pipeline

```
[ITSM / SIEM Alerts]  (ServiceNow / Sentinel / Splunk)
          |
          v
[Incident Intake: detected_at, aware_at] ---> [Dependency Graph: affected CIFs?]
          |                                              |
          v                                              v
[RTS Classification Engine] <---------------- [Criteria: clients, txns, duration,
          |                                     geo spread, data loss, economic]
   +------+---------------------+
   |                            |
[Non-major]                 [MAJOR: classified_at]
   |                            |
   v                            v
[Recurring Aggregator]   [Initial Notification <= 4h / <= 24h from awareness]
 (same root cause,              |
  >= 2 in 6 months)             v
                         [Intermediate Report <= 72h] --(status change)--> [Updated Intermediate]
                                |
                                v
                         [Final Report <= 1 month] ---> [Competent Authority Gateway]
```
