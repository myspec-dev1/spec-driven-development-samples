# Constitution: DORA ICT Risk & Operational Resilience Platform

## 1. Purpose & Mission
The DORA ICT Risk & Operational Resilience Platform is a regulatory-grade system of record that enables EU financial entities (credit institutions, investment firms, payment & e-money institutions, insurers, CASPs) to comply with Regulation (EU) 2022/2554 (Digital Operational Resilience Act), applicable since 17 January 2025. It unifies the ICT asset & risk inventory, major ICT-related incident classification and 3-stage supervisory reporting, the ITS-format Register of Information on ICT third-party service providers, concentration-risk analytics for critical or important functions (CIFs), digital operational resilience testing including Threat-Led Penetration Testing (TLPT, TIBER-EU), and tested exit strategies.

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: Regulatory Clocks Are Never Missed
- Every major ICT-related incident drives three hard deadlines computed from immutable timestamps (`detected_at`, `classified_at`, report `submitted_at`):
  $$T_{\text{initial}} = \min\left(t_{\text{classified}} + 4\text{h},\; t_{\text{aware}} + 24\text{h}\right), \quad T_{\text{intermediate}} = t_{\text{initial\_sent}} + 72\text{h}, \quad T_{\text{final}} = t_{\text{last\_intermediate}} + 1\text{ month}$$
- Escalation alerts fire at 50%, 75%, and 90% of each window; a breached deadline is recorded as a permanent compliance event and can never be silently cleared.

### Tenet 2: Deterministic, Explainable Classification
- Major-incident classification follows the RTS criteria (critical services affected, clients/counterparts/transactions, reputational impact, duration & downtime, geographical spread, data losses, economic impact) as a pure, versioned rule function.
- Identical inputs and ruleset version MUST yield an identical verdict; every verdict stores each criterion's value, threshold, and pass/fail so supervisors can replay it.

### Tenet 3: Register of Information Is a Validated Single Source of Truth
- Each contractual arrangement, ICT provider (identified by LEI, or EUID where no LEI exists), and supported function exists exactly once; exports to xBRL-CSV MUST pass all ESA validation rules before release.
- No CIF may be linked to a provider without a substitutability assessment and a documented exit strategy.

### Tenet 4: Append-Only Evidence & Auditability
- Incident timelines, submitted reports, test findings, and register snapshots are append-only and hash-chained (SHA-256); corrections are new versions referencing the prior version, never in-place edits.

### Tenet 5: Proportionality Without Loss of Rigor
- The platform supports both the full ICT risk management framework and the simplified framework for small/non-interconnected entities via configuration only, never by bypassing classification or reporting logic.

## 3. Scope Boundaries

### What We Are Building
- ICT asset, business-function, and risk inventory with CIF mapping and dependency graph.
- Major incident classification engine, deadline watchdog, and initial/intermediate/final report generator for competent authorities.
- ITS Register of Information editor with LEI validation and xBRL-CSV export package builder.
- Concentration-risk analytics, resilience testing & TLPT lifecycle tracker, and exit-strategy plan registry.

### What We Are NOT Building
- We are not a SIEM/SOC detection tool; incidents are ingested from existing monitoring and ITSM systems.
- We do not perform penetration testing or red teaming ourselves, nor do we implement ESA oversight of critical ICT third-party providers (CTPPs).
