# Implementation Tasks: Virtual Power Plant & DER Aggregation Platform

## Phase 1: Device Integration Gateway
- [ ] **TSK-VPP-01**: Build IEEE 2030.5 server for device polling (mutual-TLS PKI, LFDI/SFDI registry, `DERControl` / `DefaultDERControl` events) and CSIP aggregator client toward the utility server.
- [ ] **TSK-VPP-02**: Implement OpenADR 3.0 VTN (programs, events, reports, subscriptions) and VEN adapter for utility DR programs.
- [ ] **TSK-VPP-03**: Build SunSpec Modbus edge agent with `SunS` model discovery and read/write of Model 1, 700-series DER, and 800-series storage models.
- [ ] **TSK-VPP-04**: Create OEM cloud connectors (battery, EV charger, thermostat) mapped to the common capability model, with expiring controls and revert-to-default handling.

## Phase 2: Telemetry Ingestion & Forecasting
- [ ] **TSK-VPP-05**: Build Kafka/Flink telemetry pipeline into TimescaleDB hypertables with `STALE` / `ESTIMATED` / `OUT_OF_RANGE` quality flags.
- [ ] **TSK-VPP-06**: Implement Redis live fleet state (available kW up/down, SoC, override flag) refreshed within 10 s.
- [ ] **TSK-VPP-07**: Train LightGBM quantile models producing P10/P50/P90 flexibility per portfolio and feeder over a 48-hour horizon.
- [ ] **TSK-VPP-08**: Implement pinball-loss calibration monitor with retraining alert when P90 coverage falls below 88%.

## Phase 3: Grid Constraints & Dispatch Optimisation
- [ ] **TSK-VPP-09**: Build operating envelope intake (IEEE 2030.5 `opModExpLimW` / `opModImpLimW` and utility REST) with 30-second utility override pre-emption.
- [ ] **TSK-VPP-10**: Implement HiGHS rolling-horizon LP dispatch with ramp, SoC, customer reserve, comfort band, EV departure, and feeder envelope constraints.
- [ ] **TSK-VPP-11**: Implement greedy merit-order fallback allocator and shortfall detection with re-dispatch and operator alerting.
- [ ] **TSK-VPP-12**: Build regulation signal follower and DR event handler, plus FERC Order 2222 aggregation registry with double-counting checks.

## Phase 4: Settlement, M&V & Customer Payouts
- [ ] **TSK-VPP-13**: Implement baseline engine (10-of-10 with ±20% same-day adjustment, meter-before/meter-after) with append-only baseline snapshots.
- [ ] **TSK-VPP-14**: Build interval performance calculator and ISO settlement statement reconciliation flagging differences above 1% per trading day.
- [ ] **TSK-VPP-15**: Implement customer incentive calculator (per-event \$/kWh, capacity \$/kW, revenue share) and approved payout batch export.
- [ ] **TSK-VPP-16**: Add two-person approval and rate limiting for fleet commands above 10 MW or 5,000 devices, with signed command audit log.

## Phase 5: Scale Benchmarking & Grid-Safety Validation
- [ ] **TSK-VPP-17**: Load test 100,000 simulated devices at 50,000 samples/s; verify p99 ingest-to-queryable latency $\le 5\text{ s}$.
- [ ] **TSK-VPP-18**: Benchmark the dispatch optimiser on 100,000 devices × 12 intervals; verify solve time $\le 20\text{ s}$ and controls reaching 95% of devices within 60 s.
- [ ] **TSK-VPP-19**: Run 10,000 randomised dispatch scenarios; verify zero feeder envelope violations and zero breaches of customer backup reserve.
- [ ] **TSK-VPP-20**: Simulate a 30-minute cloud outage; verify 100% of devices revert to their default mode within $2\times$ the dispatch interval.
