# Constitution: FHIR Payer Interoperability & Electronic Prior Authorization Platform

## 1. Purpose & Mission
The FHIR Payer Interoperability & Electronic Prior Authorization Platform is a regulated health-plan integration hub that brings impacted payers (Medicare Advantage organizations, state Medicaid and CHIP programs and their managed care plans, and QHP issuers on the Federally-Facilitated Exchanges) into compliance with the CMS Interoperability and Prior Authorization Final Rule (CMS-0057-F) ahead of the 1 January 2027 API deadline. It exposes HL7 FHIR R4 Patient Access, Provider Access, Payer-to-Payer, and Prior Authorization APIs, replacing fax- and portal-driven prior authorization with an end-to-end Da Vinci CRD → DTR → PAS workflow embedded directly in provider EHRs.

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: Regulatory Decision Clocks Are Absolute
- Every prior authorization request starts an immutable decision clock at the moment of receipt. For each request $r$:
  $$t_{\text{decision}}(r) - t_{\text{received}}(r) \le \begin{cases} 72\text{ hours} & \text{if expedited} \\ 7\text{ calendar days} & \text{if standard} \end{cases}$$
- Clock state is computed server-side from the authoritative receipt timestamp; it can never be paused, reset, or back-dated by a user or downstream system.

### Tenet 2: No Silent Denials
- Every denial (full or partial) MUST carry a specific, machine-readable reason and a human-readable explanation returned on the `ClaimResponse`, never a generic "not medically necessary" placeholder.
- Pended requests MUST state exactly which documentation is missing, so the provider can act without a phone call.

### Tenet 3: Member Consent Governs Data Movement
- Payer-to-Payer exchange occurs only after an explicit, recorded member opt-in; Provider Access exchange honors member opt-out and a verified provider–member treatment relationship (attribution).
- Every disclosure is journaled with requester identity, legal basis, consent version, and resources released.

### Tenet 4: Standards-Conformant, Never Proprietary
- All APIs MUST conform to FHIR R4 (4.0.1), US Core, SMART App Launch, Bulk Data Access, and the Da Vinci CRD, DTR, and PAS implementation guides. No proprietary payload extensions may be required for a conformant EHR to complete a transaction.

### Tenet 5: Transparency by Construction
- Prior authorization metrics required for annual public reporting (approval, denial, approval-after-appeal, extension rates, and average/median decision times) are derived from the same immutable decision journal used for clock enforcement — never from a separately maintained spreadsheet.

## 3. Scope Boundaries

### What We Are Building
- FHIR R4 Patient Access, Provider Access, Payer-to-Payer, and Prior Authorization APIs with SMART on FHIR / OAuth 2.0 + OIDC authorization.
- Da Vinci CRD (CDS Hooks), DTR (Questionnaire + CQL), and PAS (`Claim/$submit`, `Claim/$inquire`) services with bidirectional X12 278 translation.
- Decision-clock orchestration, specific denial-reason catalog, and public prior authorization metrics reporting.

### What We Are NOT Building
- We do not handle prior authorization for drugs (pharmacy or medical-benefit drugs), which the rule excludes; those remain on NCPDP ePA channels.
- We do not make clinical coverage determinations ourselves — licensed reviewers and the payer's medical policy engine remain the decision authority.
