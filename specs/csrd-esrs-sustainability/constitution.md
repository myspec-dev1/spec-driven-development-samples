# Constitution: CSRD / ESRS Sustainability & Double Materiality Platform

## 1. Purpose & Mission
The CSRD / ESRS Sustainability & Double Materiality Platform enables large European enterprises and international parent organizations to fulfill mandatory statutory reporting under the EU Corporate Sustainability Due Diligence Directive (CSDDD) and Corporate Sustainability Reporting Directive (CSRD). It provides a legally defensible system of record for the European Sustainability Reporting Standards (ESRS), conducting quantitative Double Materiality Assessments (DMA), tracking Scope 1-3 GHG emissions, and exporting audited Inline XBRL (iXBRL) filings.

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: Uncompromising Double Materiality Assessment (DMA) Independence
- Materiality must be independently scored across both dimensions simultaneously:
  1. **Impact Materiality (Inside-Out)**: Scale, scope, and irremediable character of actual or potential impacts on people and the environment.
  2. **Financial Materiality (Outside-In)**: Magnitude and likelihood of financial risks or opportunities affecting cash flows, enterprise value, or access to capital.
- An ESRS topic, sub-topic, or disclosure requirement is deemed material if it meets either the impact threshold $\theta_{\text{impact}}$ OR the financial threshold $\theta_{\text{financial}}$; a high financial rating cannot cancel out an adverse environmental impact.

### Tenet 2: End-to-End Raw Evidence Audit Lineage
- Under no circumstances shall an ESRS datapoint exist without cryptographic lineage linking directly to primary raw evidence (e.g. utility bill PDF, ERP fuel invoice, travel expense report, supplier ESG questionnaire).
- Every aggregated metric must support drill-down to the originating meter reading or calculation factor with full version history.

### Tenet 3: Strict Greenhouse Gas Protocol (GHGP) Scope 1, 2, and 3 Accounting
- Carbon accounting must strictly enforce GHG Protocol standards and ISO 14064-1:
  - Scope 1: Direct emissions from owned or controlled sources.
  - Scope 2: Location-based and Market-based indirect emissions from purchased electricity/steam.
  - Scope 3: Upstream and downstream value chain activities across all 15 standard categories.
- Emission factor sets (e.g., DEFRA, IEA, eGRID, EXIOBASE) must be version-locked per reporting year; factor updates cannot silently recalculate historical baseline years.

### Tenet 4: Machine-Readable Inline XBRL (iXBRL) Semantic Fidelity
- Disclosures must be tagged directly within XHTML narratives using the official European Financial Reporting Advisory Group (EFRAG) ESRS XBRL taxonomy.
- Tagged numerical values must pass automated ESEF calculation and duplicate consistency validations prior to auditor sign-off.

### Tenet 5: Third-Party External Assurance Readiness (Limited & Reasonable Assurance)
- All calculation formulas, transformation scripts, emission factor references, and stakeholder interview scoring sheets must be exportable in an "Auditor Package" compliant with the International Standard on Assurance Engagements (ISAE 3000 / ISSA 5000).

## 3. Scope Boundaries

### What We Are Building
- Quantitative & qualitative Double Materiality Assessment (DMA) scoring matrix.
- Comprehensive ESRS datapoint tracking catalog (ESRS 2 General Disclosures, E1-E5 Environmental, S1-S4 Social, G1 Governance).
- Automated Scope 1, 2, and 3 GHG emissions calculation engine.
- Value chain supplier data collection portal with automated validation checks.
- EFRAG compliant Inline XBRL (iXBRL) digital tagging and filing export.

### What We Are NOT Building
- We do not issue third-party statutory auditor assurance opinions (we provide the software evidence platform used by statutory auditors like PwC, EY, KPMG, and Deloitte).
- We do not trade voluntary carbon offset credits or broker carbon offsets.
