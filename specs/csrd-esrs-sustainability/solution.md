# Solution Architecture: CSRD / ESRS Sustainability & Double Materiality Platform

## 1. System Architecture Overview

The platform implements an auditable, data-lineage architecture:
1. **Core Data Platform (PostgreSQL 16 & Graph Engine)**: Stores normalized sustainability metrics, organizational hierarchies, and an explicit Directed Acyclic Graph (DAG) for raw evidence lineage.
2. **GHG Protocol Carbon Calculation Engine (Python / NumPy)**: High-speed matrix multiplication calculating Scope 1, 2, and 3 emissions against versioned emission factor databases.
3. **Double Materiality Scoring Service (NestJS / TypeScript)**: Manages qualitative surveys, stakeholder weights, and real-time matrix threshold gating.
4. **iXBRL Document Packaging Service (Go / Arelle C-bindings)**: Embeds XML taxonomy tags into XHTML annual report chapters and generates validated ESEF submission packages.

```
+-----------------------------------------------------------------------------------------+
|                                    DATA INPUT SOURCES                                   |
|                                                                                         |
|   [Utility Billing Invoices]    [ERP Procurement Spend]    [Supplier Survey Portal]     |
+-----------------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                                  ACTIVITY INGESTION & OCR                               |
|                                                                                         |
|  [PDF OCR Extractor] ---> [Spend Category Mapper] ---> [Evidence S3 Storage + SHA-256]  |
+-----------------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                               CARBON & METRIC CALCULATION                               |
|                                                                                         |
|  [GHG Engine: Scope 1, 2, 3] <---> [Factor Database (DEFRA, eGRID, IEA, EXIOBASE)]     |
+-----------------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                               DOUBLE MATERIALITY & REPORTING                            |
|                                                                                         |
|  [Double Materiality Engine] ---> [Lineage DAG Graph] ---> [iXBRL / ESEF Exporter]      |
+-----------------------------------------------------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Double Materiality Topics & Scores (`dma_topics` & `dma_scores`)
```sql
CREATE TABLE dma_topics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    esrs_standard VARCHAR(8) NOT NULL, -- E1, E2, E3, E4, E5, S1, S2, S3, S4, G1
    topic_name VARCHAR(128) NOT NULL,
    sub_topic_name VARCHAR(128) NOT NULL,
    description TEXT NOT NULL
);

CREATE TABLE dma_assessments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    reporting_year INT NOT NULL,
    topic_id UUID NOT NULL REFERENCES dma_topics(id),
    -- Impact Materiality (Inside-Out)
    impact_scale INT NOT NULL CHECK (impact_scale BETWEEN 1 AND 5),
    impact_scope INT NOT NULL CHECK (impact_scope BETWEEN 1 AND 5),
    impact_irremediability INT NOT NULL CHECK (impact_irremediability BETWEEN 1 AND 5),
    impact_likelihood INT NOT NULL CHECK (impact_likelihood BETWEEN 1 AND 5),
    impact_total_score NUMERIC(6, 2) NOT NULL,
    -- Financial Materiality (Outside-In)
    financial_magnitude INT NOT NULL CHECK (financial_magnitude BETWEEN 1 AND 5),
    financial_likelihood INT NOT NULL CHECK (financial_likelihood BETWEEN 1 AND 5),
    financial_total_score NUMERIC(6, 2) NOT NULL,
    -- Gating Result
    is_material BOOLEAN NOT NULL,
    materiality_justification TEXT NOT NULL,
    assessed_by UUID NOT NULL,
    assessed_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    UNIQUE (organization_id, reporting_year, topic_id)
);
```

### 2.2 Raw Evidence Lineage Graph (`esrs_evidence_nodes` & `esrs_lineage_edges`)
```sql
CREATE TABLE esrs_evidence_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type VARCHAR(32) NOT NULL, -- RAW_INVOICE, EMISSION_FACTOR, CALCULATION_STEP, DISCLOSED_DATAPOINT
    label VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    sha256_hash VARCHAR(64) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL
);

CREATE TABLE esrs_lineage_edges (
    parent_node_id UUID NOT NULL REFERENCES esrs_evidence_nodes(id),
    child_node_id UUID NOT NULL REFERENCES esrs_evidence_nodes(id),
    transformation_type VARCHAR(64) NOT NULL, -- MULTIPLY_FACTOR, AGGREGATE_SUM, UNIT_CONVERSION
    formula TEXT,
    PRIMARY KEY (parent_node_id, child_node_id)
);
```

## 3. Double Materiality Calculation Engine
```typescript
export interface DmaInput {
  scale: number; // 1-5
  scope: number; // 1-5
  irremediability: number; // 1-5
  impactLikelihood: number; // 1-5
  financialMagnitude: number; // 1-5
  financialLikelihood: number; // 1-5
}

export function evaluateDoubleMateriality(
  input: DmaInput,
  impactThreshold: number = 18,
  financialThreshold: number = 12
): { impactScore: number; financialScore: number; isMaterial: boolean; reason: string } {
  const severity = input.scale + input.scope + input.irremediability;
  const impactScore = severity * input.impactLikelihood;
  const financialScore = input.financialMagnitude * input.financialLikelihood;

  const isImpactMaterial = impactScore >= impactThreshold;
  const isFinancialMaterial = financialScore >= financialThreshold;
  const isMaterial = isImpactMaterial || isFinancialMaterial;

  let reason = '';
  if (isImpactMaterial && isFinancialMaterial) {
    reason = 'Material on both Impact and Financial dimensions.';
  } else if (isImpactMaterial) {
    reason = 'Material on Impact dimension (Inside-Out).';
  } else if (isFinancialMaterial) {
    reason = 'Material on Financial dimension (Outside-In).';
  } else {
    reason = 'Non-material (Below both thresholds; documented omission).';
  }

  return { impactScore, financialScore, isMaterial, reason };
}
```
