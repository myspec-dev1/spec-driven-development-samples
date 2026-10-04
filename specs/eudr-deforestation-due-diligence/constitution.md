# Constitution: EUDR Deforestation Due-Diligence & Traceability Platform

## 1. Purpose & Mission
The EUDR Deforestation Due-Diligence & Traceability Platform enables operators and traders placing cattle, cocoa, coffee, oil palm, rubber, soya, and wood (and their Annex I derived products) on, or exporting them from, the EU market to comply with Regulation (EU) 2023/1115 by 30 December 2026 (large/medium operators). It captures plot-level geolocation from smallholders and estates, screens every plot against the 31 December 2020 deforestation cut-off using satellite forest-cover baselines, runs documented risk assessment and mitigation, traces commodity flows across supplier tiers and HS codes, and submits Due Diligence Statements (DDS) to the EU TRACES Information System.

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: No Plot, No Product
- Every unit of relevant commodity in a shipment must trace back to at least one geolocated plot of land with a recorded production date range. Volume with unknown origin can never be declared compliant:
  $$\forall\, b \in \text{Batches}_{\text{DDS}} : \; \text{Plots}(b) \neq \emptyset \;\land\; \sum_{p \in \text{Plots}(b)} q_{p \to b} \ge Q_b$$
- Mixing compliant volume with volume of unknown or non-compliant origin taints the entire batch; it is quarantined as `NON_COMPLIANT_BLEND` and cannot be referenced in any DDS.

### Tenet 2: Deforestation-Free Against the 31 December 2020 Cut-Off
- A plot is eligible only if no forest-to-non-forest conversion (and, for wood, no forest degradation) occurred after 31 Dec 2020, using the FAO forest definition (> 0.5 ha, trees > 5 m, canopy cover > 10%).
- Satellite screening outcomes are evidence, not verdicts: any detected loss inside a plot buffer forces `MANUAL_REVIEW`; automated auto-clearance is allowed only when detected post-2020 loss is exactly $0\text{ m}^2$.

### Tenet 3: Negligible Risk Before Submission
- A DDS may only be submitted when the documented risk assessment concludes risk is "none or negligible". Non-negligible risk must be mitigated (supplier audits, additional documents, independent surveys) and re-assessed; risk can never be overridden by a single user without dual sign-off.
- Country benchmarking (low / standard / high) adjusts assessment depth but never removes the geolocation and cut-off checks.

### Tenet 4: Immutable 5-Year Evidence Trail
- Every geolocation upload, satellite check result, risk assessment, mitigation action, and DDS (with TRACES reference and verification numbers) is stored append-only, hash-chained, and retained for a minimum of 5 years from the date of placing on the market or export.

### Tenet 5: Geodetic Precision Is Preserved End-to-End
- Coordinates are stored and transmitted in WGS84 (EPSG:4326) with at least 6 decimal places (~0.11 m at the equator); no reprojection, rounding, or simplification may alter submitted geometry.

## 3. Scope Boundaries

### What We Are Building
- Mobile/offline plot mapping and bulk GeoJSON/Shapefile/KML import with topology validation.
- Satellite deforestation screening engine against JRC GFC2020, Hansen Global Forest Change, and alert layers (GLAD, RADD).
- Risk assessment & mitigation workflow incorporating EU country benchmarking and legality evidence.
- Multi-tier chain-of-custody ledger (segregated and identity-preserved blending) keyed by HS code.
- TRACES DDS submission, amendment, withdrawal, and downstream DDS reference linking.

### What We Are NOT Building
- We are not a certification scheme or competent authority; third-party certificates (RSPO, FSC, Rainforest Alliance) are supporting evidence only and never substitute for due diligence.
- We do not perform customs declarations or classify goods; HS codes are supplied by the operator's customs broker.
