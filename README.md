# ARIA v2.0 — Terrain Intelligence Upgrade

Extension of Week 3 ARIA (river flood risk) with DEM-based terrain analysis for Hualien County shelters.

## What's New in v2.0

- **DEM terrain analysis**: MOI 20m DEM slope & elevation extraction per shelter
- **Zonal statistics**: Mean elevation + max slope within 500m buffer of each shelter
- **Slope risk classification**: LOW (0-15°) / MEDIUM (15-30°) / HIGH (>30°)
- **Composite risk scoring**: River risk + terrain risk → CRITICAL / HIGH / MEDIUM / LOW

## Data Sources

| Data | Source | Format |
|------|--------|--------|
| Rivers | OSM Overpass API | GeoPackage |
| Shelters | 消防署開放資料 | CSV |
| Townships | 國土測繪中心 TGOS | Shapefile |
| DEM | 內政部 20m DEM | GeoTIFF |

## Outputs

- `ARIA_v2.ipynb` — Full analysis notebook
- `terrain_risk_audit.json` — Per-shelter risk assessment (198 records)
- `terrain_risk_map.png` — Three-panel terrain visualization (elevation, slope, risk)

## Results Summary (Hualien County, 198 shelters)

**Terrain Risk:**
- LOW (0-15°): 87 shelters
- MEDIUM (15-30°): 29 shelters
- HIGH (>30°): 82 shelters

**Composite Risk (river + terrain):**
- CRITICAL: 66 shelters
- HIGH: 85 shelters
- MEDIUM: 39 shelters
- LOW: 8 shelters

## Requirements

```
geopandas rioxarray xarray rasterio numpy matplotlib folium
```

## Usage

```bash
source gis-env/bin/activate
jupyter notebook ARIA_v2.ipynb
```

For Colab: upload notebook + mount Google Drive with DEM file.
