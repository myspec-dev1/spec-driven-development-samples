# Solution Architecture: Carbon Removal MRV & Credit Registry Platform

## 1. System Architecture Overview

The system separates evidence-heavy MRV processing from the strictly consistent unit ledger:
1. **MRV Ingestion Gateway (Go / Kafka)**: Receives sensor telemetry (flow meters, CO2 purity analyzers), lab assay certificates (biochar H/C$_{\text{org}}$, rock weathering soil cation assays), and storage injection reports; validates signatures and writes raw evidence to WORM object storage (S3 Object Lock).
2. **Quantification Engine (Python / NumPy / Ray)**: Executes versioned methodology modules computing gross removals, baseline, lifecycle (LCA) emissions, leakage, and Monte Carlo uncertainty discounts with fully reproducible seeds and pinned emission factor versions.
3. **Verification Workflow Service (NestJS / PostgreSQL / Temporal)**: Orchestrates VVB assignment, conflict checks, CAR/CL/FAR findings, and qualified-signature verification statements as durable workflows.
4. **Unit Ledger Core (Go / PostgreSQL SERIALIZABLE)**: Holds serial ranges, accounts, and the hash-chained event journal; enforces conservation of units, range splitting, and dual-approval issuance.
5. **Integrity & Article 6 Service (Go / PostGIS)**: Performs geospatial and feedstock-batch overlap checks against external registry feeds and tracks letters of authorization and corresponding adjustment status.
6. **Public Registry API (Go / Redis / CDN)**: Read-only replica serving projects, issuances, retirements, and serial lookups with cursor pagination.

```
+-----------------------------------------------------------------------------------------+
|                                   MRV DATA SOURCES                                      |
|  [DAC Plant SCADA]  [Biochar Lab Assays]  [ERW Soil Sampling]  [Storage Operator Reports]|
+-----------------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                         MRV INGESTION GATEWAY (Go / Kafka)                              |
|  [Signature Check] ---> [Schema & Unit Validation] ---> [WORM Evidence Store (S3 Lock)] |
+-----------------------------------------------------------------------------------------+
                                          | (Kafka: mrv.evidence.accepted)
                                          v
+-----------------------------------------------------------------------------------------+
|                     QUANTIFICATION ENGINE (Python / Ray)                                |
|  [Gross - Baseline] ---> [LCA + Leakage] ---> [Monte Carlo 90% CI] ---> [Buffer Score]  |
+-----------------------------------------------------------------------------------------+
                                          |
                                          v
+---------------------------------------+   +---------------------------------------------+
|   VERIFICATION WORKFLOW (Temporal)    |   |   INTEGRITY & ARTICLE 6 SERVICE (PostGIS)   |
|  - VVB Conflict & Rotation Checks     |-->|  - Cross-Registry Overlap Detection         |
|  - CAR / CL / FAR Findings            |   |  - LoA & Corresponding Adjustment Tracking  |
+---------------------------------------+   +---------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                       UNIT LEDGER CORE (Go / PostgreSQL SERIALIZABLE)                   |
|  [Issue Ranges] [Split & Transfer] [Retire w/ Beneficiary] [Cancel] [Hash-Chain Journal]|
+-----------------------------------------------------------------------------------------+
                                          | (Logical Replication)
                                          v
+-----------------------------------------------------------------------------------------+
|                  PUBLIC REGISTRY API (Read Replica / Redis / CDN)                       |
+-----------------------------------------------------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Monitoring Period Quantification (`monitoring_periods`)
```sql
CREATE TABLE monitoring_periods (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id UUID NOT NULL REFERENCES projects(id),
    methodology_version VARCHAR(32) NOT NULL,
    pathway VARCHAR(16) NOT NULL CHECK (pathway IN ('BIOCHAR', 'DACCS', 'BECCS', 'ERW')),
    period_start TIMESTAMPTZ NOT NULL,
    period_end TIMESTAMPTZ NOT NULL,
    gross_removals_t NUMERIC(18, 6) NOT NULL CHECK (gross_removals_t >= 0),
    baseline_removals_t NUMERIC(18, 6) NOT NULL CHECK (baseline_removals_t >= 0),
    lifecycle_emissions_t NUMERIC(18, 6) NOT NULL CHECK (lifecycle_emissions_t >= 0),
    leakage_emissions_t NUMERIC(18, 6) NOT NULL CHECK (leakage_emissions_t >= 0),
    carried_deficit_t NUMERIC(18, 6) NOT NULL DEFAULT 0 CHECK (carried_deficit_t >= 0),
    uncertainty_discount NUMERIC(5, 4) NOT NULL CHECK (uncertainty_discount BETWEEN 0 AND 0.5),
    buffer_rate NUMERIC(5, 4) NOT NULL CHECK (buffer_rate BETWEEN 0 AND 0.5),
    issuable_units BIGINT NOT NULL DEFAULT 0 CHECK (issuable_units >= 0),
    mc_seed BIGINT NOT NULL,
    status VARCHAR(24) NOT NULL DEFAULT 'DRAFT' CHECK (status IN ('DRAFT', 'UNDER_VERIFICATION', 'VERIFIED', 'ISSUED', 'REJECTED')),
    verification_statement_id UUID,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CHECK (period_end > period_start),
    EXCLUDE USING gist (project_id WITH =, tstzrange(period_start, period_end) WITH &&)
);
```

### 2.2 Serialized Unit Ranges (`unit_ranges`)
```sql
CREATE TABLE unit_ranges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    issuance_id UUID NOT NULL REFERENCES issuances(id),
    serial_start BIGINT NOT NULL CHECK (serial_start > 0),
    serial_end BIGINT NOT NULL,
    quantity_t BIGINT GENERATED ALWAYS AS (serial_end - serial_start + 1) STORED,
    vintage SMALLINT NOT NULL CHECK (vintage BETWEEN 2000 AND 2100),
    account_id UUID NOT NULL REFERENCES accounts(id),
    state VARCHAR(16) NOT NULL CHECK (state IN ('ACTIVE', 'RETIRED', 'CANCELLED', 'BUFFER')),
    authorization_status VARCHAR(40) NOT NULL DEFAULT 'UNAUTHORIZED_MITIGATION_CONTRIBUTION'
        CHECK (authorization_status IN ('UNAUTHORIZED_MITIGATION_CONTRIBUTION', 'AUTHORIZED_NDC', 'AUTHORIZED_OIMP')),
    ca_status VARCHAR(16) NOT NULL DEFAULT 'NOT_REQUIRED' CHECK (ca_status IN ('NOT_REQUIRED', 'PENDING', 'APPLIED')),
    retirement_beneficiary VARCHAR(256),
    retirement_purpose VARCHAR(16) CHECK (retirement_purpose IN ('VOLUNTARY', 'CORSIA', 'NDC')),
    superseded_at TIMESTAMPTZ, -- set when split; history is never deleted
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CHECK (serial_end >= serial_start),
    CHECK (state <> 'RETIRED' OR retirement_beneficiary IS NOT NULL),
    CHECK (retirement_purpose NOT IN ('CORSIA', 'NDC') OR ca_status = 'APPLIED'),
    EXCLUDE USING gist (issuance_id WITH =, int8range(serial_start, serial_end, '[]') WITH &&) WHERE (superseded_at IS NULL)
);
```

## 3. Net Removal Quantification & Issuance Sizing
```python
from dataclasses import dataclass
from decimal import Decimal, ROUND_CEILING, ROUND_FLOOR
import numpy as np

@dataclass(frozen=True)
class PeriodInputs:
    gross_samples: np.ndarray      # Monte Carlo draws of gross removals (tCO2e)
    baseline_samples: np.ndarray
    lifecycle_samples: np.ndarray  # ISO 14040/14044 cradle-to-grave emissions
    leakage_samples: np.ndarray
    carried_deficit_t: Decimal
    buffer_rate: Decimal           # from non-permanence risk tool

MAX_DISCOUNT = Decimal("0.50")

def size_issuance(p: PeriodInputs) -> dict:
    net = p.gross_samples - p.baseline_samples - p.lifecycle_samples - p.leakage_samples
    mean = Decimal(str(float(np.mean(net))))
    lo, hi = np.percentile(net, [5, 95])  # 90% confidence interval
    if mean <= 0:
        return {"issuable_units": 0, "buffer_units": 0, "new_deficit_t": p.carried_deficit_t - mean}

    rel_half_width = Decimal(str(float((hi - lo) / 2))) / mean
    discount = max(Decimal("0"), rel_half_width - Decimal("0.10"))
    if discount > MAX_DISCOUNT:
        raise ValueError(f"uncertainty too high: half-width {rel_half_width:.2%} exceeds 60%")

    conservative = mean * (1 - discount) - p.carried_deficit_t
    if conservative <= 0:
        return {"issuable_units": 0, "buffer_units": 0, "new_deficit_t": -conservative}

    total_units = int(conservative.to_integral_value(rounding=ROUND_FLOOR))  # 1 unit = 1 tCO2e
    # Buffer rounds UP so the pool never receives less than the risk rate
    buffer_units = int((Decimal(total_units) * p.buffer_rate).to_integral_value(rounding=ROUND_CEILING))
    return {
        "issuable_units": total_units - buffer_units,
        "buffer_units": buffer_units,
        "uncertainty_discount": discount,
        "new_deficit_t": Decimal("0"),
    }
```
