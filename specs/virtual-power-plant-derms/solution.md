# Solution Architecture: Virtual Power Plant & DER Aggregation Platform

## 1. System Architecture Overview

The system is split into a protocol edge, a streaming data plane, and a market-facing control plane:
1. **Device Integration Gateway (Go)**: Protocol adapters for IEEE 2030.5 / CSIP (mutual-TLS server polled by devices, LFDI-keyed; aggregator client toward the utility DERMS), OpenADR 3.0 (VTN and VEN), OEM cloud APIs, and a lightweight SunSpec Modbus edge agent on site gateways. Translates every device into a common capability and control model.
2. **Telemetry Pipeline (Kafka / Flink / TimescaleDB)**: Partitions telemetry by device ID, applies data quality flags, maintains live fleet state in Redis, and stores 5-minute rollups in TimescaleDB hypertables.
3. **Forecasting Service (Python / LightGBM quantile regression)**: Produces P10/P50/P90 flexibility forecasts per portfolio and feeder from weather, SoC trajectories, and historical availability.
4. **Dispatch Optimiser (Python / HiGHS LP solver)**: Solves rolling-horizon allocation per interval subject to ramp, SoC, customer reserve, and feeder envelope constraints; a fast greedy allocator serves as fallback if the solver exceeds its time budget.
5. **Market & DERMS Interface (NestJS)**: Submits bids and offers to ISO/RTO market systems, receives awards and regulation signals, and ingests utility operating envelopes and overrides.
6. **Settlement & M&V Engine (Go / PostgreSQL)**: Computes baselines, interval performance, ISO statement reconciliation, and customer payout batches.

```
+-----------------------------------------------------------------------------------------+
|                         MARKET, UTILITY & EXTERNAL INPUTS                               |
|                                                                                         |
|  [ISO/RTO Awards & Reg Signals]   [Utility DERMS Operating Envelopes]   [Weather APIs]  |
+-----------------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                               CONTROL PLANE                                             |
|                                                                                         |
|  [Forecast P10/P50/P90] ---> [Bid Sizing @ P90] ---> [Dispatch Optimiser (HiGHS LP)]   |
|                                                              |                          |
|                                          [Envelope / Reserve / Ramp Constraint Guard]   |
+-----------------------------------------------------------------------------------------+
                                          | (Kafka: vpp.device.controls)
                                          v
+-----------------------------------------------------------------------------------------+
|                          DEVICE INTEGRATION GATEWAY (Go)                                |
|                                                                                         |
|  [IEEE 2030.5 / CSIP]   [OpenADR 3.0 VTN/VEN]   [SunSpec Modbus Edge]   [OEM Cloud APIs]|
+-----------------------------------------------------------------------------------------+
                                          | (Kafka: vpp.telemetry.raw)
                                          v
+---------------------------------------+   +---------------------------------------------+
|         TELEMETRY & FLEET STATE       |   |              SETTLEMENT & M&V               |
|  - TimescaleDB 5-min hypertables      |   |  - 10-of-10 Baselines & Performance         |
|  - Redis live device state            |   |  - ISO Statement Reconciliation & Payouts   |
+---------------------------------------+   +---------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Device Telemetry Hypertable (`der_telemetry`)
```sql
CREATE TABLE der_telemetry (
    device_id UUID NOT NULL REFERENCES der_devices(id),
    feeder_id VARCHAR(64) NOT NULL,
    sample_time TIMESTAMPTZ NOT NULL,
    received_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    active_power_kw NUMERIC(10, 3) NOT NULL, -- + export / discharge, - import / charge
    reactive_power_kvar NUMERIC(10, 3),
    soc_pct NUMERIC(5, 2) CHECK (soc_pct BETWEEN 0 AND 100),
    voltage_v NUMERIC(7, 2),
    frequency_hz NUMERIC(6, 3),
    operating_mode VARCHAR(32) NOT NULL CHECK (operating_mode IN ('SELF_CONSUMPTION', 'DISPATCHED', 'CURTAILED', 'OVERRIDE', 'OFFLINE')),
    quality_flag VARCHAR(16) NOT NULL DEFAULT 'VALID' CHECK (quality_flag IN ('VALID', 'ESTIMATED', 'STALE', 'OUT_OF_RANGE')),
    PRIMARY KEY (device_id, sample_time)
);
SELECT create_hypertable('der_telemetry', 'sample_time', chunk_time_interval => INTERVAL '1 day');
```

### 2.2 Dispatch Instructions & Envelope Audit (`dispatch_instructions`)
```sql
CREATE TABLE dispatch_instructions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    portfolio_id UUID NOT NULL REFERENCES portfolios(id),
    device_id UUID NOT NULL REFERENCES der_devices(id),
    feeder_id VARCHAR(64) NOT NULL,
    market_product VARCHAR(32) NOT NULL CHECK (market_product IN ('ENERGY_DA', 'ENERGY_RT', 'REGULATION', 'SPINNING_RESERVE', 'DEMAND_RESPONSE', 'UTILITY_OVERRIDE')),
    interval_start TIMESTAMPTZ NOT NULL,
    interval_end TIMESTAMPTZ NOT NULL CHECK (interval_end > interval_start),
    target_kw NUMERIC(10, 3) NOT NULL,
    feeder_oe_min_kw NUMERIC(12, 3) NOT NULL,
    feeder_oe_max_kw NUMERIC(12, 3) NOT NULL CHECK (feeder_oe_max_kw >= feeder_oe_min_kw),
    soc_floor_pct NUMERIC(5, 2) NOT NULL CHECK (soc_floor_pct BETWEEN 0 AND 100),
    protocol VARCHAR(16) NOT NULL CHECK (protocol IN ('IEEE_2030_5', 'OPENADR_3', 'SUNSPEC_MODBUS', 'OEM_API')),
    expires_at TIMESTAMPTZ NOT NULL CHECK (expires_at > interval_start),
    status VARCHAR(16) NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING', 'SENT', 'ACKED', 'REJECTED', 'EXPIRED', 'SUPERSEDED')),
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);
```

## 3. Envelope-Aware Dispatch Allocation (Greedy Fallback)
```python
from dataclasses import dataclass
from collections import defaultdict

@dataclass
class DeviceState:
    device_id: str
    feeder_id: str
    p_prev_kw: float          # current output (+ discharge / export)
    p_max_kw: float           # max discharge at inverter rating
    ramp_kw_per_min: float
    soc_kwh: float
    reserve_kwh: float        # customer backup reserve floor
    cost_per_kwh: float       # degradation + incentive cost

def allocate_discharge(target_kw: float, devices: list[DeviceState],
                       feeder_oe_max_kw: dict[str, float],
                       feeder_baseline_kw: dict[str, float],
                       interval_min: float = 5.0) -> tuple[dict[str, float], float]:
    """Merit-order allocation honouring ramp, SoC reserve, and feeder operating envelopes."""
    hours = interval_min / 60.0
    feeder_headroom = {f: feeder_oe_max_kw[f] - feeder_baseline_kw.get(f, 0.0) for f in feeder_oe_max_kw}
    plan: dict[str, float] = defaultdict(float)
    remaining = target_kw

    for d in sorted(devices, key=lambda x: x.cost_per_kwh):
        if remaining <= 1e-3:
            break
        energy_cap_kw = max(0.0, (d.soc_kwh - d.reserve_kwh) / hours)    # Tenet 2: never breach reserve
        ramp_cap_kw = d.p_prev_kw + d.ramp_kw_per_min * interval_min    # FR-DSP-02 ramp limit
        envelope_cap_kw = max(0.0, feeder_headroom.get(d.feeder_id, 0.0))  # Tenet 1: feeder OE
        p = min(d.p_max_kw, energy_cap_kw, ramp_cap_kw, envelope_cap_kw, remaining)
        if p <= 0:
            continue
        plan[d.device_id] = round(p, 3)
        feeder_headroom[d.feeder_id] -= p
        remaining -= p

    shortfall_kw = max(0.0, remaining)  # FR-DSP-05: report, never exceed envelope to close the gap
    return dict(plan), shortfall_kw
```
