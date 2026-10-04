# Implementation Tasks: EUDR Deforestation Due-Diligence & Traceability Platform

## Phase 1: Plot Geolocation Capture & Validation
- [ ] **TSK-EDR-01**: Build offline-first field mapping app with GNSS boundary walking, HDOP $\le 2.0$ accuracy gating, and 30-day sync queue.
- [ ] **TSK-EDR-02**: Implement GeoJSON / KML / Shapefile / CSV bulk importer normalizing to WGS84 (EPSG:4326) with 6-decimal precision enforcement.
- [ ] **TSK-EDR-03**: Implement point-vs-polygon 4 ha rule, OGC topology validation, and cross-supplier plot overlap detection ($> 5\%$).
- [ ] **TSK-EDR-04**: Build plausibility checks (water/urban masks, country boundary, commodity yield-per-hectare ceilings).

## Phase 2: Satellite Deforestation Screening Engine
- [ ] **TSK-EDR-05**: Ingest JRC GFC2020 and Hansen Global Forest Change as Cloud-Optimized GeoTIFFs resampled to a shared 10 m grid with dataset versioning.
- [ ] **TSK-EDR-06**: Implement Dask-based zonal statistics computing post-31-Dec-2020 loss per plot (30 m point buffer) with `CLEAR` / `MANUAL_REVIEW` / `FLAGGED` classification.
- [ ] **TSK-EDR-07**: Build weekly RADD and GLAD alert re-screening job with 24-hour analyst notification.
- [ ] **TSK-EDR-08**: Build GIS analyst review console with before/after imagery overlays and evidence snapshot persistence.

## Phase 3: Risk Assessment & Chain of Custody
- [ ] **TSK-EDR-09**: Implement versioned country benchmarking table (low / standard / high) and simplified due diligence path for low-risk origins.
- [ ] **TSK-EDR-10**: Build multi-criteria risk scoring (deforestation, legality, FPIC, supplier history, tier complexity) and mitigation case workflow with dual sign-off.
- [ ] **TSK-EDR-11**: Implement segregated and controlled-blend custody ledger rejecting any lot containing unknown-origin or non-`CLEAR` inputs.
- [ ] **TSK-EDR-12**: Implement HS-code transformation conversion factors and volume conservation checks ($\le 2\%$ tolerance) with recursive lot-to-plot lineage resolution.

## Phase 4: DDS Submission & TRACES Integration
- [ ] **TSK-EDR-13**: Build DDS assembler (operator EORI, HS code, net mass, supplementary units, country, geolocation) with SHA-256 payload hashing.
- [ ] **TSK-EDR-14**: Implement TRACES gateway for submit / amend / withdraw with idempotency keys and reference + verification number persistence.
- [ ] **TSK-EDR-15**: Implement upstream DDS reference validation and customs declaration export of DDS reference numbers.
- [ ] **TSK-EDR-16**: Build S3 Object Lock WORM evidence vault with hash-chained audit log, 5-year retention, and competent-authority export package.

## Phase 5: Scale Benchmarking & Compliance Validation
- [ ] **TSK-EDR-17**: Benchmark satellite screening of 5,000,000 plots across 7 commodities; verify throughput $\ge 1{,}000{,}000$ plots/hour.
- [ ] **TSK-EDR-18**: Validate screening accuracy against 2,000 analyst-labelled plots; verify recall $\ge 95\%$ for post-2020 deforestation $> 0.5\text{ ha}$.
- [ ] **TSK-EDR-19**: Verify DDS assembly for a 50,000-plot shipment completes in $\le 60\text{ s}$ and 10,000 retried TRACES submissions produce zero duplicate DDSs.
- [ ] **TSK-EDR-20**: Run lineage audit over 1,000,000 lots across 6 tiers; verify 100% resolve to geolocated plots in $\le 2\text{ s}$ each with zero volume-conservation violations.
