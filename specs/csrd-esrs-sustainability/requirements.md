# Requirements Specification: CSRD / ESRS Sustainability & Double Materiality Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **Chief Sustainability Officer / ESG Lead (ACT-CSO)**: Configures materiality thresholds, oversees stakeholder surveys, and signs off on report narratives.
- **Carbon Accounting Specialist (ACT-ENG)**: Manages utility data feeds, selects emission factor libraries, and computes GHG footprints.
- **Supplier / Supply Chain Partner (ACT-SPL)**: Enters Scope 3 questionnaire data and uploads supplier certificates via the external portal.
- **Internal Audit & Risk Director (ACT-RSK)**: Evaluates climate financial risks (physical and transition risks) aligned with TCFD.
- **External Statutory Assurance Auditor (ACT-AUD)**: External auditor reviewing documentation, evidence lineage, and XBRL tags.

---

## 2. Functional Requirements

### 2.1 Double Materiality Assessment (DMA) Matrix Engine (FR-DMA)
- **FR-DMA-01 (Two-Dimensional Topic Evaluation)**: The system MUST maintain the complete EFRAG ESRS topic tree (10 topical standards: E1 Climate, E2 Pollution, E3 Water, E4 Biodiversity, E5 Circular Economy, S1 Own Workforce, S2 Value Chain Workers, S3 Affected Communities, S4 Consumers, G1 Business Conduct).
- **FR-DMA-02 (Impact Materiality Scoring)**: For every sub-topic, the system MUST compute Impact Severity:
  $$\text{Severity} = \text{Scale (1..5)} + \text{Scope (1..5)} + \text{Irremediability (1..5)}$$
  and Impact Score:
  $$\text{Score}_{\text{impact}} = \text{Severity} \times \text{Likelihood (1..5)}$$
- **FR-DMA-03 (Financial Materiality Scoring)**: The system MUST calculate Financial Materiality based on anticipated monetary effect on cash flows, EBITDA, or asset impairment:
  $$\text{Score}_{\text{financial}} = \text{Magnitude (1..5)} \times \text{Likelihood (1..5)}$$
- **FR-DMA-04 (Automated Disclosure Scoping)**: When a topic crosses either $\text{Score}_{\text{impact}} \ge \theta_{\text{impact}}$ OR $\text{Score}_{\text{financial}} \ge \theta_{\text{financial}}$, the system MUST automatically mark all corresponding ESRS Disclosure Requirements (DR) as `MANDATORY_IN_SCOPE` with a traceable justification memo.

### 2.2 Scope 1, 2, and 3 Carbon Accounting Engine (FR-GHG)
- **FR-GHG-01 (Multi-Source Activity Ingestion)**: The system MUST ingest activity data from multiple sources:
  - Electricity & natural gas utility bills (via OCR / EDI / Energy Star API).
  - Fleet telematics and fuel cards (liters/gallons).
  - Corporate travel booking systems (flight itineraries, hotel nights).
  - ERP purchasing spend (Scope 3 Category 1: Purchased Goods and Services).
- **FR-GHG-02 (Emission Factor Library Management)**: The system MUST maintain versioned emission factor databases:
  - UK DEFRA / DESNZ
  - US EPA eGRID
  - IEA Global Grid Factors
  - Ecoinvent & EXIOBASE EEIO spend factors
- **FR-GHG-03 (Dual Scope 2 Reporting)**: The engine MUST simultaneously compute both Location-Based (grid average) and Market-Based (contractual instruments, Renewable Energy Certificates - RECs, Guarantees of Origin - GOs) Scope 2 emissions in metric tons $\text{CO}_2\text{e}$.

### 2.3 Evidence Lineage & Audit Trail (FR-EVD)
- **FR-EVD-01 (Granular Datapoint Lineage Graph)**: For every reported numerical figure (e.g. Total Scope 1 = 14,280 tCO2e), the system MUST render an interactive lineage graph tracing back to every individual fuel receipt, conversion factor, and calculation formula.
- **FR-EVD-02 (Document Storage & OCR Extraction)**: Raw evidence documents (invoices, meter logs, PDF statements) MUST be permanently stored with SHA-256 content hashes and text OCR search indexing.

### 2.4 EFRAG-Compliant Inline XBRL (iXBRL) Export (FR-XBR)
- **FR-XBR-01 (ESEF / ESRS Taxonomy Tagging)**: The system MUST provide an interactive tagging interface allowing users to map report text and numerical tables to official EFRAG ESRS XBRL taxonomy tags.
- **FR-XBR-02 (Statutory Package Generation)**: The system MUST export an ESEF-compliant ZIP archive containing XHTML files with embedded iXBRL tags, taxonomy schema extensions, and calculation linkbases.
- **FR-XBR-03 (Automated Taxonomy Validator)**: The exporter MUST run Arelle / ESEF validation rules, detecting duplicate tags, missing dimensional contexts, or calculation inconsistencies before report finalization.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Regulatory Defensibility & Assurance Readiness (NFR-REG)
- **NFR-REG-01 (ISAE 3000 / ISSA 5000 Package)**: System MUST export a consolidated auditor dossier including calculation methodologies, factor citations, and approval signatures in $< 30\text{ seconds}$.
- **NFR-REG-02 (Immutable Historical Baselines)**: Historical base year emissions cannot be updated directly; adjustments must follow GHG Protocol base-year recalculation policies with documented structural change thresholds ($> 5\%$).

### 3.2 Security, Privacy & Multitenancy (NFR-SEC)
- **NFR-SEC-01 (Confidentiality & Role-Based Scoping)**: Supply chain vendor submissions and sensitive financial risk forecasts MUST be isolated via strict role permissions; suppliers can only view their own questionnaire data.
- **NFR-SEC-02 (WORM Evidence Protection)**: Evidence files linked to published CSRD annual reports MUST be locked in compliance-mode WORM storage for a statutory minimum of 10 years.

### 3.3 Scalability & Data Processing (NFR-PERF)
- **NFR-PERF-01 (Spend-Based Scope 3 Processing)**: Processing and mapping 250,000 ERP procurement ledger rows against EEIO emission factor matrices MUST complete in $\le 60\text{ seconds}$.

---

## 4. Double Materiality Assessment Flow

```
+-------------------------------------------------------------------------------+
|                       IDENTIFY SUSTAINABILITY MATTERS                         |
|             (ESRS Topical Standards: E1-E5, S1-S4, G1 + Entity-Specific)      |
+-------------------------------------------------------------------------------+
                                        |
                    +-------------------+-------------------+
                    |                                       |
                    v                                       v
+---------------------------------------+   +-----------------------------------+
|      IMPACT MATERIALITY SCORING       |   |   FINANCIAL MATERIALITY SCORING   |
|                                       |   |                                   |
| - Scale (1..5)                        |   | - Magnitude of Financial Risk/    |
| - Scope (1..5)                        |   |   Opportunity (1..5)              |
| - Irremediability (1..5)              |   | - Likelihood of Impact (1..5)     |
| - Likelihood (1..5)                   |   |                                   |
|                                       |   |                                   |
| Score = Severity x Likelihood         |   | Score = Magnitude x Likelihood    |
+---------------------------------------+   +-----------------------------------+
                    |                                       |
                    +-------------------+-------------------+
                                        |
                                        v
+-------------------------------------------------------------------------------+
|                        DOUBLE MATERIALITY THRESHOLD GATING                    |
|                                                                               |
|             Is (Score_impact >= Theta_I) OR (Score_financial >= Theta_F)?     |
+-------------------------------------------------------------------------------+
                         |                                      |
                     [YES]                                     [NO]
                         |                                      |
                         v                                      v
+---------------------------------------+   +-----------------------------------+
|      IN-SCOPE DISCLOSURE (ESRS)       |   |       OUT-OF-SCOPE (DOCUMENTED)   |
| - Activate Data Collection Workflows  |   | - Generate Materiality Exclusion  |
| - Mandate Raw Evidence Attachment    |   |   Explanation Memo for Auditors   |
| - Map to EFRAG XBRL Taxonomy Tags     |   |                                   |
+---------------------------------------+   +-----------------------------------+
```
