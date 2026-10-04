# Constitution: Virtual Power Plant & DER Aggregation Platform

## 1. Purpose & Mission
The Virtual Power Plant (VPP) & DER Aggregation Platform turns hundreds of thousands of small, customer-owned Distributed Energy Resources (residential and C&I batteries, rooftop solar inverters, heat pumps, and EV chargers) into reliable, dispatchable portfolios. It integrates devices over IEEE 2030.5 (CSIP), OpenADR 3.0, SunSpec Modbus, and OEM cloud APIs; forecasts probabilistic flexibility; dispatches portfolios against wholesale energy, ancillary service, and demand response commitments (including FERC Order 2222 DER aggregations); and settles performance through auditable baselines and customer incentive payouts, all without breaching the distribution grid limits set by the host utility's DERMS.

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: The Grid Operating Envelope Is Absolute
- For every feeder (or other network node) $f$ and every dispatch interval $t$, the aggregate net active power of all enrolled devices must remain inside the operating envelope published by the utility DERMS:
  $$\text{OE}^{\min}_f(t) \;\le\; \sum_{d \in \mathcal{D}_f} P_d(t) \;\le\; \text{OE}^{\max}_f(t)$$
- Market revenue never overrides an envelope. If a market award cannot be met inside the envelopes, the shortfall is declared to the market operator; the envelope is never violated.

### Tenet 2: Customer Comfort & Backup Reserve Come First
- No dispatch may drive a battery below its customer-configured backup reserve ($SoC_d(t) \ge SoC^{\text{reserve}}_d$), push a heat-pump zone outside its comfort band, or leave an EV below its departure-time energy target.
- Customer opt-outs and manual overrides take effect at the device in $\le 60\text{ seconds}$ and are honoured without penalty to the customer.

### Tenet 3: Fail-Safe Local Autonomy
- Loss of cloud connectivity must never leave a device stuck in a dispatched state. Every control carries an explicit expiry, and devices revert to their local default mode (self-consumption, or the IEEE 2030.5 `DefaultDERControl`) when it lapses.
- Device-level IEEE 1547-2018 grid-support functions (voltage/frequency ride-through, volt-var) are never disabled by the platform.

### Tenet 4: Performance Is Measured, Not Assumed
- Every settled kW and kWh is backed by interval meter or device telemetry and a documented baseline methodology (e.g. 10-of-10). Raw telemetry, baselines, and adjustments are stored append-only so any settlement statement can be recomputed byte-for-byte.

### Tenet 5: Commitments Are Probabilistic
- Bids and capacity offers are sized from the forecast P90 availability (the volume exceeded with 90% probability), not the P50 expectation, so the portfolio can deliver its awards even with heavy device dropout.

## 3. Scope Boundaries

### What We Are Building
- Multi-protocol device integration gateway (IEEE 2030.5 / CSIP client, OpenADR 3.0 VTN & VEN, SunSpec Modbus edge agent, OEM cloud connectors).
- Telemetry ingestion for 100,000+ devices, P10/P50/P90 availability forecasting, and optimisation-based portfolio dispatch.
- Utility operating-envelope intake, M&V baseline engine, market settlement reconciliation, and customer incentive payouts.

### What We Are NOT Building
- We are not the utility DERMS, ADMS, or SCADA system; we consume their envelopes and signals and do not run distribution power flow.
- We do not build device firmware or inverter control loops, and we do not act as a retail energy supplier or bill retail tariffs.
