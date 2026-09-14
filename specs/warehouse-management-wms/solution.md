# Solution Architecture: Warehouse Management System (WMS) & Multimodal Freight Logistics Platform

## 1. System Architecture Overview

The platform uses an event-sourced, domain-driven microservices architecture:
1. **Warehouse Operations Core (WMS)**: Go-based microservices managing high-concurrency bin inventory transactions, directed pick/putaway path engines, and wave generation.
2. **Multimodal Freight Service**: Node.js / TypeScript orchestrator managing carrier state machines, booking engines, and AIS / GPS tracking event streams.
3. **EDI Translation & AS2 Gateway**: Specialized Java / Spring Boot integration engine handling AS2 protocol connections, S/MIME encryption, and bidirectional ANSI X12 / UN-EDIFACT schema mapping.
4. **Demurrage Watchdog Service**: Real-time cron and Kafka stream consumer monitoring terminal milestones and calculating penalty risks.

```
+-----------------------------------------------------------------------------------------+
|                                  WAREHOUSE OPERATORS                                    |
|                                                                                         |
|   [Zebra Android Handhelds]     [Forklift Tablets]     [Packing Station Touchscreens]   |
+-----------------------------------------------------------------------------------------+
                                          | (Fast WebSocket / gRPC over Wi-Fi 6)
                                          v
+-----------------------------------------------------------------------------------------+
|                                    WMS CORE ENGINE (Go)                                 |
|                                                                                         |
|  [Inventory Engine] ---> [Putaway Solver] ---> [Wave / Batch Planner] ---> [LPN Service]|
+-----------------------------------------------------------------------------------------+
                                          | (Kafka Events: wms.orders.allocated)
                                          v
+-----------------------------------------------------------------------------------------+
|                              FREIGHT & LOGISTICS ENGINE                                 |
|                                                                                         |
|  [Multi-Leg State Machine] <---> [Demurrage Penalty Clock] <---> [Customs HS Classifier]|
+-----------------------------------------------------------------------------------------+
                                          ^
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                            EDI TRANSLATION & AS2 GATEWAY                                |
|                                                                                         |
|  [AS2 Endpoint] <---> [ANSI X12 (204/214/304)] <---> [EDIFACT (IFTMIN/IFTSTA/DESADV)]   |
+-----------------------------------------------------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Warehouse Bin Locations & Inventory Balances
```sql
CREATE TABLE warehouse_locations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    warehouse_code VARCHAR(16) NOT NULL,
    zone VARCHAR(16) NOT NULL,
    aisle VARCHAR(8) NOT NULL,
    rack VARCHAR(8) NOT NULL,
    shelf VARCHAR(8) NOT NULL,
    bin VARCHAR(8) NOT NULL,
    barcode VARCHAR(64) UNIQUE NOT NULL,
    max_weight_kg NUMERIC(10, 2) NOT NULL,
    is_bonded BOOLEAN DEFAULT false NOT NULL,
    is_active BOOLEAN DEFAULT true NOT NULL
);

CREATE TABLE inventory_license_plates (
    lpn VARCHAR(64) PRIMARY KEY, -- GS1-128 Serial Shipping Container Code (SSCC)
    location_id UUID NOT NULL REFERENCES warehouse_locations(id),
    sku VARCHAR(64) NOT NULL,
    lot_number VARCHAR(64) NOT NULL,
    quantity_on_hand INT NOT NULL CHECK (quantity_on_hand >= 0),
    quantity_allocated INT DEFAULT 0 NOT NULL CHECK (quantity_allocated <= quantity_on_hand),
    expiration_date DATE,
    received_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);
```

### 2.2 Container & Demurrage Monitoring (`freight_containers`)
```sql
CREATE TABLE freight_containers (
    container_number VARCHAR(11) PRIMARY KEY, -- ISO 6346 (e.g. MSKU1234567)
    shipment_id UUID NOT NULL,
    carrier_scac VARCHAR(4) NOT NULL, -- Standard Carrier Alpha Code
    terminal_code VARCHAR(16) NOT NULL,
    vessel_discharge_time TIMESTAMPTZ,
    free_time_days INT NOT NULL,
    free_time_expiry_time TIMESTAMPTZ,
    daily_demurrage_rate_usd NUMERIC(10, 2) NOT NULL,
    gate_out_time TIMESTAMPTZ,
    status VARCHAR(32) NOT NULL CHECK (status IN ('ON_VESSEL', 'DISCHARGED', 'GATED_OUT', 'DETENTION', 'RETURNED'))
);
```

## 3. Demurrage Calculation Algorithm
```typescript
interface DemurrageStatus {
  isAccruing: boolean;
  hoursRemaining: number;
  accruedCostUSD: number;
  urgencyLevel: 'NORMAL' | 'WARNING' | 'CRITICAL' | 'PENALTY_ACCRUING';
}

export function computeDemurrage(container: ContainerRecord, currentTime: Date): DemurrageStatus {
  if (!container.vesselDischargeTime || container.gateOutTime) {
    return { isAccruing: false, hoursRemaining: 0, accruedCostUSD: 0, urgencyLevel: 'NORMAL' };
  }

  const freeTimeMs = container.freeTimeDays * 24 * 60 * 60 * 1000;
  const expiryTime = new Date(container.vesselDischargeTime.getTime() + freeTimeMs);
  const diffMs = expiryTime.getTime() - currentTime.getTime();

  if (diffMs > 0) {
    const hoursRemaining = diffMs / (1000 * 60 * 60);
    const urgency = hoursRemaining <= 24 ? 'CRITICAL' : hoursRemaining <= 48 ? 'WARNING' : 'NORMAL';
    return { isAccruing: false, hoursRemaining, accruedCostUSD: 0, urgencyLevel: urgency };
  } else {
    const overdueDays = Math.ceil(Math.abs(diffMs) / (1000 * 60 * 60 * 24));
    const accruedCost = overdueDays * container.dailyDemurrageRateUsd;
    return { isAccruing: true, hoursRemaining: 0, accruedCostUSD: accruedCost, urgencyLevel: 'PENALTY_ACCRUING' };
  }
}
```
