# Requirements Specification: Virtual Power Plant & DER Aggregation Platform

## 1. System Overview & Actors

### 1.1 Actors & Personas
- **VPP Portfolio Operator (ACT-OPS)**: Monitors live portfolio output, approves market bids, and handles dispatch exceptions and device fleet alarms.
- **Energy Trader / Market Scheduler (ACT-TRD)**: Submits day-ahead and real-time bids, capacity offers, and ancillary service offers to the ISO/RTO or DR program administrator.
- **Utility DERMS Operator (ACT-UTL)**: Publishes feeder operating envelopes, hosting-capacity limits, and curtailment overrides; reviews FERC Order 2222 aggregation registrations.
- **Enrolled Customer / Prosumer (ACT-CUS)**: Owns the battery, inverter, heat pump, or EV charger; sets backup reserve, comfort band, and EV departure targets; receives incentive payouts.
- **Settlement & M&V Analyst (ACT-SET)**: Reviews baseline calculations, reconciles ISO settlement statements, and approves customer incentive runs.
- **Device Integration Gateway (ACT-GW)**: Automated agent that speaks IEEE 2030.5, OpenADR 3.0, SunSpec Modbus, and OEM cloud APIs.

---

## 2. Functional Requirements

### 2.1 Device Integration & Enrollment (FR-DEV)
- **FR-DEV-01 (IEEE 2030.5 / CSIP Server & Aggregator Client)**: Toward devices, the system MUST operate an IEEE 2030.5 server (DER head-end) that enrolled inverters and site gateways poll for `DERControl` / `DefaultDERControl` events (`opModFixedW`, `opModExpLimW`, `opModConnect`, `opModEnergize`), authenticating each device certificate over TLS 1.2 (`TLS_ECDHE_ECDSA_WITH_AES_128_CCM_8`) and identifying it by LFDI/SFDI. Toward the utility, it MUST act as a CSIP aggregator client to the utility's 2030.5 server.
- **FR-DEV-02 (OpenADR 3.0 VTN & VEN)**: The system MUST expose an OpenADR 3.0 VTN (REST/JSON programs, events, reports, subscriptions) toward downstream C&I sites, and act as a VEN toward utility DR programs, acknowledging events and returning telemetry reports.
- **FR-DEV-03 (SunSpec Modbus Edge Agent)**: An on-site edge agent MUST read and write SunSpec Information Models over Modbus TCP/RTU, including Model 1 (Common), the 700-series DER models aligned to IEEE 1547-2018, and the 800-series storage models, discovering models from the `SunS` base register.
- **FR-DEV-04 (OEM Cloud Connectors)**: The system MUST integrate vendor cloud APIs for batteries, EV chargers (including OCPP 2.0.1 via charge point operators), and smart thermostats through a common capability model (`max_charge_kw`, `max_discharge_kw`, `usable_kwh`, `ramp_kw_per_min`).
- **FR-DEV-05 (Customer Enrollment & Constraints)**: Enrollment MUST capture meter ID, feeder/transformer mapping, program consent, backup reserve (default 20% SoC), heat-pump comfort band (e.g. $\pm 1.5^\circ\text{C}$), and EV departure time + target kWh.

### 2.2 Telemetry Ingestion & Fleet State (FR-TEL)
- **FR-TEL-01 (High-Volume Telemetry)**: The system MUST ingest active/reactive power, SoC, voltage, frequency, and operating mode from $\ge 100{,}000$ devices at a 1–5 minute cadence (2–4 s for devices enrolled in frequency regulation).
- **FR-TEL-02 (Data Quality Flags)**: Every sample MUST be stamped with device and receive timestamps, and flagged `ESTIMATED`, `STALE` (older than $2\times$ the expected cadence), or `OUT_OF_RANGE`; flagged samples are excluded from settlement unless replaced by revenue-meter data.
- **FR-TEL-03 (Live Fleet State)**: The system MUST maintain a per-device state (online, available kW up/down, SoC, active control, override flag) refreshed within 10 s of the latest sample.

### 2.3 Forecasting & Availability (FR-FCST)
- **FR-FCST-01 (Probabilistic Availability)**: The system MUST produce P10/P50/P90 forecasts of available upward and downward flexibility per portfolio and per feeder at 5-minute granularity for a 48-hour horizon, updated at least hourly.
- **FR-FCST-02 (Forecast Inputs)**: Forecasts MUST combine solar irradiance and temperature forecasts, historical device availability, SoC trajectories, EV plug-in probabilities, and customer opt-out rates.
- **FR-FCST-03 (Forecast Calibration)**: Quantile calibration MUST be tracked by pinball loss; the P90 volume MUST be met or exceeded in $\ge 88\%$ of intervals over a rolling 30 days or the model is flagged for retraining.

### 2.4 Market Participation & Dispatch Optimisation (FR-DSP)
- **FR-DSP-01 (FERC Order 2222 Aggregations)**: The system MUST register and manage DER aggregations meeting each ISO's Order 2222 rules (minimum size no greater than 100 kW), tracking per-resource eligibility, distribution utility review status, and no double-counting across retail DR and wholesale programs.
- **FR-DSP-02 (Optimal Dispatch)**: For each interval the optimiser MUST allocate portfolio targets across devices by minimising cost (degradation, customer incentive, shortfall penalty) subject to:
  $$\sum_{d} P_d(t) = P^{\text{award}}(t), \quad P_d^{\min} \le P_d(t) \le P_d^{\max}, \quad |P_d(t) - P_d(t-1)| \le R_d \,\Delta t$$
  plus SoC limits, customer reserves, and feeder operating envelopes.
- **FR-DSP-03 (Ancillary & Frequency Response)**: The system MUST follow ISO regulation signals within the ISO's required response time and support autonomous frequency-watt response configured on the device for fast frequency response products.
- **FR-DSP-04 (Demand Response Events)**: The system MUST accept DR event notifications (OpenADR, utility API, or ISO dispatch) and send device controls at least 5 minutes before event start where notice allows.
- **FR-DSP-05 (Shortfall Handling)**: If delivered output deviates from the award by more than 5% for 2 consecutive intervals, the system MUST re-dispatch reserve devices and, if still short, alert ACT-OPS and ACT-TRD to update the market schedule.

### 2.5 Grid Constraints & DERMS Coordination (FR-GRID)
- **FR-GRID-01 (Operating Envelope Intake)**: The system MUST ingest static hosting-capacity limits and dynamic operating envelopes (import/export kW per feeder, transformer, or site, at 5–30 minute resolution) via IEEE 2030.5 (`opModExpLimW` / `opModImpLimW`, CSIP-AUS style) or utility REST APIs.
- **FR-GRID-02 (Envelope Pre-Check)**: Every dispatch plan MUST be validated against current envelopes before release; any violating plan is clipped and the clipped volume logged.
- **FR-GRID-03 (Utility Override Priority)**: A utility curtailment or emergency override MUST pre-empt all market dispatch on the affected feeder within 30 seconds.

### 2.6 Settlement, M&V & Customer Payouts (FR-STL)
- **FR-STL-01 (Baseline Calculation)**: The system MUST compute customer baselines using configurable methods, including 10-of-10 (average of the 10 most recent eligible non-event days at the same interval) with an optional same-day additive adjustment capped at $\pm 20\%$, and meter-before/meter-after for batteries.
- **FR-STL-02 (Performance Calculation)**: Delivered energy per interval MUST be calculated as $E^{\text{delivered}}(t) = \text{Baseline}(t) - \text{Metered}(t)$ and aggregated to 5-minute or hourly ISO settlement intervals.
- **FR-STL-03 (ISO Statement Reconciliation)**: The system MUST reconcile internal performance against ISO/utility settlement statements and flag differences $> 1\%$ per trading day.
- **FR-STL-04 (Customer Incentive Payouts)**: The system MUST calculate per-customer incentives (per-event \$/kWh, monthly capacity \$/kW, or revenue share) and export approved payout batches to the payments provider with an itemised statement per customer.

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Performance & Scale (NFR-PERF)
- **NFR-PERF-01 (Telemetry Throughput)**: Ingestion MUST sustain 50,000 samples/second with p99 ingest-to-queryable latency $\le 5\text{ s}$.
- **NFR-PERF-02 (Dispatch Solve Time)**: The optimiser MUST solve a 100,000-device, 12-interval dispatch problem in $\le 20\text{ s}$, and push device controls to 95% of targeted devices within 60 s.

### 3.2 Security & Grid Cybersecurity (NFR-SEC)
- **NFR-SEC-01 (Device PKI)**: All device sessions MUST use mutual TLS with per-device certificates (IEEE 2030.5 PKI for CSIP devices); control commands are signed and logged with operator or service identity.
- **NFR-SEC-02 (Fleet Control Safeguards)**: Commands affecting $> 10\text{ MW}$ or $> 5{,}000$ devices MUST require two-person approval or a pre-approved market award, and are rate-limited to prevent a compromised account from triggering synchronised load swings.

### 3.3 Reliability & Fail-Safe (NFR-REL)
- **NFR-REL-01 (Control Plane Availability)**: The dispatch control plane MUST achieve 99.95% monthly availability with active-active deployment across 2 regions.
- **NFR-REL-02 (Expiring Controls)**: Every device control MUST carry an expiry of $\le 2\times$ the dispatch interval; on expiry or loss of connectivity devices revert to their local default mode.

---

## 4. Award-to-Settlement Dispatch Pipeline

```
[ISO / RTO Awards & Signals]   [Utility DERMS Envelopes]   [Weather & Fleet Telemetry]
  (DA / RT / Regulation / DR)    (Feeder OE kW, overrides)    (P / SoC / plug-in state)
            |                               |                            |
            |                               |                            v
            |                               |              [P10/P50/P90 Availability Forecast]
            |                               |                            |
            +---------------+---------------+-------------+--------------+
                            |                             |
                            v                             v
                 [Bid Sizing @ P90]  ----->  [Dispatch Optimiser (per interval)]
                                                          |
                                         [Envelope & Customer Constraint Check]
                                                          |
                         +--------------------------------+---------------+
                         |                                |               |
                         v                                v               v
                 [IEEE 2030.5 / CSIP]            [OpenADR 3.0 VTN]   [SunSpec / OEM APIs]
                         |                                |               |
                         +----------------+---------------+---------------+
                                          |
                                          v
                             [Telemetry & Revenue Meter Data]
                                          |
                                          v
                     [Baseline (10-of-10) & M&V Performance Calc]
                                          |
                         +----------------+----------------+
                         |                                 |
                         v                                 v
              [ISO Settlement Reconciliation]     [Customer Incentive Payouts]
```
