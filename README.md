# ARIA v2.0 — Integrated Impact Auditor (Terrain Intelligence Upgrade)

Extension of Week 3 ARIA (river flood risk) with DEM-based terrain analysis for Hualien County shelters.

## What's New in v2.0

- **DEM terrain analysis**: MOI 20m DEM slope & elevation extraction per shelter
- **Zonal statistics**: Mean elevation + max slope within 500m buffer of each shelter
- **Slope risk classification**: LOW (0-15°) / MEDIUM (15-30°) / HIGH (>30°)
- **Composite risk logic**: River distance + slope + elevation → CRITICAL / HIGH / MEDIUM / LOW
- **Environment variables**: Thresholds loaded from `.env` via `python-dotenv`

## Data Sources

| Data | Source | Format |
|------|--------|--------|
| Rivers | OSM Overpass API | GeoPackage |
| Shelters | 消防署開放資料 | CSV |
| Townships | 國土測繪中心 TGOS | Shapefile |
| DEM | 內政部 20m DEM | GeoTIFF |

## Outputs

- `ARIA_v2.ipynb` — Full analysis notebook with Captain's Log markdown cells
- `terrain_risk_audit.json` — Per-shelter risk assessment with risk_level, mean_elevation, max_slope, river_distance_category
- `terrain_risk_map.png` — DEM hillshade + shelter composite risk overlay
- `top10_risk_scatter.png` — Top 10 high-risk shelters: slope vs elevation scatter plot

## Composite Risk Logic

Thresholds from `.env`: `SLOPE_THRESHOLD=30`, `ELEVATION_LOW=50`, `BUFFER_HIGH=500`

| Level | Condition |
|-------|-----------|
| CRITICAL | River < 500m **AND** max slope > 30° |
| HIGH | River < 500m **OR** max slope > 30° |
| MEDIUM | River < 1000m **AND** mean elevation < 50m |
| LOW | All others |

## Requirements

```
geopandas rioxarray rasterio numpy matplotlib python-dotenv
```

## Usage

```bash
source gis-env/bin/activate
jupyter notebook ARIA_v2.ipynb
```

For Colab: upload notebook + mount Google Drive with DEM file.

---

## AI 診斷日誌 (AI Diagnostic Log)

### 問題 1：Zonal Stats 回傳 NaN

**症狀**：早期版本中 `mean_elevation` 和 `max_slope` 全部為 NaN。

**根因**：避難所座標為 EPSG:4326（經緯度），DEM 為 EPSG:3826（TWD97 TM2）。直接用 4326 的 buffer 去 mask 3826 的 raster，導致 buffer 完全沒有覆蓋到任何 pixel。

**解法**：在做 zonal stats 之前，先將 shelter GeoDataFrame reproject 到 DEM 的 CRS：
```python
shelters_dem_crs = shelters_h.to_crs(dem_crs_rio)
shelter_buffers = shelters_dem_crs.copy()
shelter_buffers['geometry'] = shelter_buffers.geometry.buffer(500)
```

**黃金法則**：永遠 reproject vector 去對齊 raster，不要反過來（reproject raster 會 resample 像素、耗記憶體、且改變原始值）。

### 問題 2：DEM 太大導致記憶體不足

**症狀**：全台 DEM（>500MB）在 Colab 上直接 `open_rasterio()` 後記憶體爆掉。

**解法**：先用鄉鎮界 dissolve 成縣界，加上 1000m buffer（確保邊緣避難所的 500m 緩衝區不會超出 DEM），再 `rio.clip()` 裁切：
```python
county_buffered = county_boundary.buffer(1000)
dem_clip = rioxarray.open_rasterio(DEM_PATH).rio.clip(county_buffered.geometry, county_buffered.crs)
```

Pre-lab 已提供預裁切的 `dem_20m_hualien.tif`，省去這一步。

### 問題 3：坡度計算結果不合理（全部 < 1°）

**症狀**：花蓮縣多山地區，坡度最大值卻只有 0.8°。

**根因**：`np.gradient(dem, 1)` 預設 spacing=1，但 DEM 像素間距是 20m。spacing 太小導致梯度被高估後又因為 arctan 壓縮而失真。

**解法**：spacing 必須與 DEM 解析度一致：
```python
pixel_size = abs(dem_transform.a)  # 20.0
dy, dx = np.gradient(dem_data, pixel_size)
slope = np.degrees(np.arctan(np.sqrt(dx**2 + dy**2)))
```
