# Constitution: Warehouse Management System (WMS) & Multimodal Freight Logistics Platform

## 1. Purpose & Mission
The Warehouse Management System (WMS) and Multimodal Freight Logistics Platform orchestrates high-velocity fulfillment centers, cross-docking hubs, and international freight forwarding across ocean, air, rail, and drayage legs. It eliminates demurrage/detention penalty fees, bridges legacy EDI carrier protocols with modern cloud APIs, and guarantees real-time physical inventory conservation across bonded and non-bonded warehouse zones.

## 2. Non-Negotiable Core Invariants (Tenets)

### Tenet 1: Single Source of Truth for Physical Bin Location
- Under no operational circumstance shall an inventory unit exist without an explicit 3D warehouse address:
  $$\text{Location} = (\text{Zone}, \\ \text{Aisle}, \\ \text{Rack}, \\ \text{Shelf}, \\ \text{Bin})$$
- Directed pick, putaway, and replenishment workflows are deterministic and path-optimized; blind staging without location barcode scanning is strictly prohibited.

### Tenet 2: Multi-Leg Shipment State Machine Determinism
- A multimodal shipment consists of an ordered sequence of discrete transit legs (e.g. `Drayage Pick -> Rail Inbound -> Ocean Vessel -> Customs Hold -> Rail Outbound -> Final Delivery`).
- Each leg has strict handover milestones. A downstream leg cannot commence without confirmed physical custody transfer and bill of lading (BOL/Waybill) reconciliation.

### Tenet 3: Demurrage & Detention Zero-Tolerance Watchdog
- Any container landed at a marine terminal or railhead must be tracked against free-time expiration clocks with minute-level precision.
- Escalation warnings must trigger at 50%, 75%, and 90% of allowable free time to prevent costly carrier detention or port demurrage surcharges.

### Tenet 4: Bi-Directional EDI & API Parity
- The platform maintains full bidirectional protocol bridges between modern JSON/REST/GraphQL interfaces and legacy supply chain EDI standards:
  - ANSI X12: `204` (Motor Load Tender), `214` (Transportation Status), `304` (Ocean Shipping Instructions).
  - EDIFACT: `IFTMIN` (Instruction), `IFTSTA` (Status), `DESADV` (Despatch Advice).
- Every translated EDI transmission must maintain raw file payload logs for legal dispute resolution.

### Tenet 5: Customs & Tariff Classification Auditability
- Goods undergoing import/export declaration must have verified Harmonized System (HS) tariff codes, commercial invoices, and export control classifications before clearance dispatch. Changes to tariff mappings are recorded with customs broker license signatures.

## 3. Scope Boundaries

### What We Are Building
- High-velocity warehouse picking (wave, batch, zone pick) and slotting optimization.
- Container tracking, marine terminal appointment scheduling, and demurrage calculators.
- Multimodal rate quoting, load tendering, and dynamic freight carrier allocation.
- Bi-directional EDI ANSI X12 & UN/EDIFACT parser and validator engine.
- Automated customs filing preparation (ACE / Automated Commercial Environment XML).

### What We Are NOT Building
- We do not manufacture automated conveyor belts, AGVs (Automated Guided Vehicles), or AS/RS robotic hardware (we integrate via standard VDI 4250 / VDA 5050 and REST/PLC interfaces).
- We do not act as the licensed legal customs authority or take title to physical freight.
