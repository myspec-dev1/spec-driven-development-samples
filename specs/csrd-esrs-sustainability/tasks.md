# Implementation Tasks: CSRD / ESRS Sustainability & Double Materiality Platform

## Phase 1: EFRAG ESRS Taxonomy & Double Materiality Engine
- [ ] **TSK-ESG-01**: Seed database with full EFRAG ESRS topical standard taxonomy (E1-E5, S1-S4, G1, and ESRS 2).
- [ ] **TSK-ESG-02**: Implement Double Materiality Assessment (DMA) scoring algorithm and threshold gating matrix.
- [ ] **TSK-ESG-03**: Build dynamic Disclosure Requirement (DR) scoping service generating audit justification memos.
- [ ] **TSK-ESG-04**: Implement Stakeholder Consultation survey portal with weighted scoring aggregations.

## Phase 2: Scope 1, 2, and 3 Carbon Accounting Engine
- [ ] **TSK-ESG-05**: Seed and version-control emission factor libraries (DEFRA 2025/2026, EPA eGRID, IEA, EXIOBASE).
- [ ] **TSK-ESG-06**: Implement automated utility bill PDF OCR parser extracting kWh, therms, and invoice dates.
- [ ] **TSK-ESG-07**: Build dual Scope 2 calculation engine (Location-Based vs. Market-Based with REC/GO matching).
- [ ] **TSK-ESG-08**: Implement Scope 3 spend-based EEIO calculation engine mapping ERP procurement codes to factor categories.

## Phase 3: Raw Evidence Lineage DAG & Document Vault
- [ ] **TSK-ESG-09**: Implement Directed Acyclic Graph (DAG) schema linking disclosed numerical values to underlying invoices.
- [ ] **TSK-ESG-10**: Build immutable S3 Object Lock document storage pipeline with SHA-256 integrity verification.
- [ ] **TSK-ESG-11**: Create interactive lineage visualizer in frontend allowing auditors to click any metric and view the calculation formula.

## Phase 4: EFRAG Inline XBRL (iXBRL) & ESEF Packaging
- [ ] **TSK-ESG-12**: Integrate official EFRAG XBRL taxonomy schemas into digital tagging interface.
- [ ] **TSK-ESG-13**: Build ESEF ZIP packager combining XHTML annual report chapters, iXBRL tags, and taxonomy linkbases.
- [ ] **TSK-ESG-14**: Embed Arelle validation engine to detect duplicate facts, calculation deviations, and schema violations.

## Phase 5: External Assurance Audit Room (ISAE 3000 / ISSA 5000)
- [ ] **TSK-ESG-15**: Build "Auditor Inspection Workspace" with one-click export of the complete ISAE 3000 assurance dossier.
- [ ] **TSK-ESG-16**: Implement base-year recalculation policy engine enforcing documented threshold adjustments ($> 5\%$).
- [ ] **TSK-ESG-17**: Benchmark spend-based Scope 3 calculator with 250,000 synthetic ERP purchasing rows; verify execution in $< 60\text{s}$.
