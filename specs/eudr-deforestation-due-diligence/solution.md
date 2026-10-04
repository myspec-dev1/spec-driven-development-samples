# Solution Architecture: EUDR Deforestation Due-Diligence & Traceability Platform

## 1. System Architecture Overview

The platform is built for geodetic precision, auditable evidence, and multi-tier supply-chain scale:
1. **Field Capture App (Kotlin Multiplatform / SQLite + SpatiaLite)**: Offline-first GNSS plot walking with accuracy gating (HDOP $\le 2.0$), farmer consent capture, and delta sync.
2. **Geo Validation & Ingestion Service (Python / Shapely / GEOS)**: Parses GeoJSON, KML, and Shapefile, enforces point-vs-polygon rules, OGC validity, and 6-decimal precision, then writes to PostGIS.
3. **Satellite Screening Engine (Python / Rasterio / Dask on Kubernetes)**: Zonal statistics over Cloud-Optimized GeoTIFF mosaics (JRC GFC2020, Hansen GFC, RADD/GLAD alerts) stored in S3, with tile-level caching.
4. **Risk & Chain-of-Custody Service (Go / PostgreSQL 16 + PostGIS)**: Computes risk scores, applies country benchmarks, enforces volume conservation across HS-code transformations, and resolves lot-to-plot lineage via recursive graph queries.
5. **TRACES Gateway (Go)**: Signs and submits DDS payloads to the EU Information System, stores reference/verification numbers, and handles retries with idempotency keys.
6. **Evidence Vault (S3 Object Lock / WORM)**: Hash-chained archive of every artefact retained for 5 years.

```
+-----------------------------------------------------------------------------------------+
|                                    DATA CAPTURE SOURCES                                 |
|                                                                                         |
|  [Field Mapping App (GNSS)]    [Supplier Portal Bulk Upload]    [ERP Lots & Shipments]  |
+-----------------------------------------------------------------------------------------+
                                          |
                                          v
+-----------------------------------------------------------------------------------------+
|                        GEO VALIDATION & INGESTION (Python / PostGIS)                    |
|                                                                                         |
|  [Format Parser] ---> [Point/Polygon 4 ha Rule] ---> [Topology + Plausibility Checks]   |
+-----------------------------------------------------------------------------------------+
                                          | (Kafka: eudr.plot.validated)
                                          v
+-----------------------------------------------------------------------------------------+
|                          SATELLITE SCREENING ENGINE (Dask / COG)                        |
|                                                                                         |
|  [JRC GFC2020 Forest Mask] ---> [Hansen Loss 2021+] ---> [RADD / GLAD Weekly Alerts]    |
+-----------------------------------------------------------------------------------------+
                                          |
                     +--------------------+--------------------+
                     |                                         |
                     v                                         v
+---------------------------------------+   +---------------------------------------------+
|     RISK & CHAIN-OF-CUSTODY (Go)      |   |          TRACES GATEWAY & EVIDENCE VAULT    |
|  - Country Benchmark (Low/Std/High)   |   |  - DDS Submit / Amend / Withdraw            |
|  - Mitigation Cases & Dual Sign-off   |   |  - Reference # + Verification # Store       |
|  - Lot-to-Plot Lineage (HS codes)     |   |  - S3 Object Lock WORM (5-year retention)   |
+---------------------------------------+   +---------------------------------------------+
```

## 2. Core Data Models (PostgreSQL DDL)

### 2.1 Production Plots & Screening Results (`eudr_plots`)
```sql
CREATE TABLE eudr_plots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id UUID NOT NULL,
    commodity VARCHAR(16) NOT NULL CHECK (commodity IN ('CATTLE', 'COCOA', 'COFFEE', 'OIL_PALM', 'RUBBER', 'SOYA', 'WOOD')),
    country_iso2 CHAR(2) NOT NULL,
    geom GEOMETRY(GEOMETRY, 4326) NOT NULL,
    area_ha NUMERIC(12, 4) NOT NULL CHECK (area_ha > 0),
    production_start DATE NOT NULL,
    production_end DATE NOT NULL CHECK (production_end >= production_start),
    screening_status VARCHAR(16) NOT NULL DEFAULT 'PENDING' CHECK (screening_status IN ('PENDING', 'CLEAR', 'MANUAL_REVIEW', 'FLAGGED', 'EXCLUDED')),
    post_cutoff_loss_ha NUMERIC(12, 4),
    baseline_dataset_version VARCHAR(64),
    screened_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT clock_timestamp() NOT NULL,
    CHECK (GeometryType(geom) IN ('POINT', 'POLYGON', 'MULTIPOLYGON')),
    CHECK (commodity = 'CATTLE' OR area_ha <= 4 OR GeometryType(geom) IN ('POLYGON', 'MULTIPOLYGON')),
    CHECK (ST_IsValid(geom))
);
CREATE INDEX idx_eudr_plots_geom ON eudr_plots USING GIST (geom);
```

### 2.2 Due Diligence Statements (`eudr_dds`)
```sql
CREATE TABLE eudr_dds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_eori VARCHAR(17) NOT NULL,
    activity_type VARCHAR(8) NOT NULL CHECK (activity_type IN ('IMPORT', 'EXPORT', 'DOMESTIC')),
    hs_code VARCHAR(10) NOT NULL,
    net_mass_kg NUMERIC(18, 3) NOT NULL CHECK (net_mass_kg > 0),
    supplementary_units NUMERIC(18, 3),
    risk_conclusion VARCHAR(16) NOT NULL CHECK (risk_conclusion IN ('NEGLIGIBLE', 'SIMPLIFIED_LOW')),
    status VARCHAR(16) NOT NULL DEFAULT 'DRAFT' CHECK (status IN ('DRAFT', 'SUBMITTED', 'AVAILABLE', 'REJECTED', 'WITHDRAWN')),
    traces_reference_number VARCHAR(32) UNIQUE,
    traces_verification_number VARCHAR(32),
    idempotency_key UUID NOT NULL UNIQUE,
    payload_sha256 CHAR(64) NOT NULL,
    signed_by UUID NOT NULL,
    submitted_at TIMESTAMPTZ,
    retain_until DATE NOT NULL,
    CHECK (status <> 'AVAILABLE' OR (traces_reference_number IS NOT NULL AND traces_verification_number IS NOT NULL)),
    CHECK (submitted_at IS NULL OR retain_until >= (submitted_at + INTERVAL '5 years')::DATE)
);
```

## 3. Post-Cut-Off Deforestation Screening Algorithm
```python
import numpy as np
import rasterio
from rasterio.mask import mask
from shapely.geometry import shape, mapping
POINT_BUFFER_M = 30.0
REVIEW_THRESHOLD_HA = 0.5

def screen_plot(geojson: dict, gfc2020_path: str, lossyear_path: str) -> dict:
    """Both COGs are pre-resampled onto the same 10 m EPSG:4326 grid (nearest neighbour)."""
    geom = shape(geojson)
    if geom.geom_type == "Point":
        # Approximate metric buffer in degrees at plot latitude (WGS84 kept for storage)
        deg = POINT_BUFFER_M / (111_320 * np.cos(np.radians(geom.y)))
        geom = geom.buffer(deg)

    with rasterio.open(gfc2020_path) as forest_src, rasterio.open(lossyear_path) as loss_src:
        forest, _ = mask(forest_src, [mapping(geom)], crop=True, nodata=0)
        loss, transform = mask(loss_src, [mapping(geom)], crop=True, nodata=0)
    assert forest.shape == loss.shape, "baseline and loss rasters must share one grid"

    # Hansen lossyear encodes 1..N as 2001..2000+N; post-cut-off loss is year >= 2021 (value >= 21)
    forest_2020 = forest[0] == 1
    post_cutoff = (loss[0] >= 21) & forest_2020

    lat = geom.centroid.y
    px_w_m = abs(transform.a) * 111_320 * np.cos(np.radians(lat))
    px_h_m = abs(transform.e) * 110_574
    loss_ha = float(post_cutoff.sum()) * px_w_m * px_h_m / 10_000

    status = "CLEAR" if loss_ha == 0 else ("MANUAL_REVIEW" if loss_ha <= REVIEW_THRESHOLD_HA else "FLAGGED")

    return {
        "status": status,
        "post_cutoff_loss_ha": round(loss_ha, 4),
        "forest_2020_pixels": int(forest_2020.sum()),
    }
```
