# Solution Architecture: Modular Cloud ERP & Shop Floor MES Platform

## 1. System Architecture Overview

The solution implements an event-driven, hybrid cloud-edge topology:
1. **Cloud ERP Core**: PostgreSQL 16+ with partitioned ledgers, Node.js / Go microservices, and GraphQL / REST APIs for finance, procurement, and enterprise planning.
2. **Plant Edge Nodes**: Embedded Kubernetes (k3s) on-premise industrial gateways running local Mosquitto / EMQX MQTT brokers, OPC-UA protocol bridges, and SQLite / DuckDB buffer caches.
3. **Time-Series Telemetry Engine**: TimescaleDB / ClickHouse for real-time sensor metrics and OEE calculation.

```
+-----------------------------------------------------------------------------------------+
|                                    CLOUD ERP CORE                                       |
|                                                                                         |
|  [Finance Service]       [Inventory Engine]       [BOM / Routing]       [MRP Engine]    |
|         \                        |                       |                     /        |
|          +-----------------------+-----------------------+--------------------+         |
|                                          |                                              |
|                               [Transactional PostgreSQL]                                |
|                                (Partitioned Append-Only)                                |
+-----------------------------------------------------------------------------------------+
                                          ^
                                          | (TLS 1.3 / gRPC Sync Protocol)
                                          v
+-----------------------------------------------------------------------------------------+
|                               PLANT EDGE GATEWAY (k3s)                                  |
|                                                                                         |
|  [Local MES Buffer Engine] <---> [EMQX Broker] <---> [OPC-UA / Modbus Bridge]           |
|            |                           |                             |                  |
|            v                           v                             v                  |
|  [Rugged Touch Terminals]    [Cycle Counter Sensors]         [CNC / PLC Controllers]    |
+-----------------------------------------------------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Double-Entry General Ledger (`gl_journal_entries` & `gl_lines`)
```sql
CREATE TABLE gl_journal_entries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entry_number VARCHAR(64) UNIQUE NOT NULL,
    fiscal_year INT NOT NULL,
    fiscal_period INT NOT NULL,
    posting_date DATE NOT NULL,
    entity_id UUID NOT NULL,
    description TEXT,
    status VARCHAR(32) NOT NULL CHECK (status IN ('DRAFT', 'POSTED', 'REVERSED')),
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

CREATE TABLE gl_lines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_id UUID NOT NULL REFERENCES gl_journal_entries(id) ON DELETE RESTRICT,
    account_code VARCHAR(32) NOT NULL,
    debit_amount NUMERIC(18, 4) DEFAULT 0.0000 NOT NULL,
    credit_amount NUMERIC(18, 4) DEFAULT 0.0000 NOT NULL,
    currency VARCHAR(3) NOT NULL,
    department_id VARCHAR(32),
    cost_center VARCHAR(32),
    CONSTRAINT check_positive_amounts CHECK (debit_amount >= 0 AND credit_amount >= 0),
    CONSTRAINT check_either_debit_or_credit CHECK (
        (debit_amount > 0 AND credit_amount = 0) OR 
        (credit_amount > 0 AND debit_amount = 0)
    )
);

-- Invariant Enforcement via Trigger
CREATE OR REPLACE FUNCTION verify_journal_balance() RETURNS TRIGGER AS $$
DECLARE
    v_total_debit NUMERIC(18,4);
    v_total_credit NUMERIC(18,4);
BEGIN
    SELECT COALESCE(SUM(debit_amount), 0), COALESCE(SUM(credit_amount), 0)
    INTO v_total_debit, v_total_credit
    FROM gl_lines WHERE journal_id = NEW.journal_id;

    IF v_total_debit != v_total_credit THEN
        RAISE EXCEPTION 'ERR_LEDGER_UNBALANCED: Debits (%) != Credits (%)', v_total_debit, v_total_credit;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 2.2 Multi-Level BOM Structure (`bom_headers` & `bom_items`)
```sql
CREATE TABLE bom_headers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    part_id UUID NOT NULL,
    revision VARCHAR(16) NOT NULL,
    effective_date DATE NOT NULL,
    is_active BOOLEAN DEFAULT true NOT NULL,
    UNIQUE (part_id, revision)
);

CREATE TABLE bom_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bom_id UUID NOT NULL REFERENCES bom_headers(id),
    component_part_id UUID NOT NULL,
    quantity_required NUMERIC(12, 4) NOT NULL,
    scrap_factor NUMERIC(5, 4) DEFAULT 0.0000 NOT NULL,
    operation_sequence INT NOT NULL
);
```

## 3. Real-Time OEE Calculation Pipeline
1. Machine PLC emits JSON payload via MQTT to edge topic `plant/{plant_id}/workcenter/{wc_id}/telemetry`:
   ```json
   {
     "timestamp": 1789360000,
     "spindle_rpm": 12400,
     "part_count_pulse": 1,
     "status": "RUNNING",
     "scrap_pulse": 0
   }
   ```
2. Edge gateway ingests and aggregates into rolling 1-minute buckets.
3. TimescaleDB continuous aggregates compute:
   - $\text{Availability} = \frac{\text{Actual Operating Time}}{\text{Planned Production Time}}$
   - $\text{Performance} = \frac{\text{Total Parts Produced} \times \text{Ideal Cycle Time}}{\text{Operating Time}}$
   - $\text{Quality} = \frac{\text{Good Parts}}{\text{Total Parts Produced}}$
4. Results are published to operator station WebSockets for immediate visual feedback.
