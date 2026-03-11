# GPS Jamming & NOAA-20 VIIRS — Technical Summary

> Codebase: `Shadowbroker-downstream` | Generated: 2026-03-11

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      BACKEND (Python/FastAPI)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Fast-tier Scheduler (every 60s)                                │
│  └─ fetch_flights() / fetch_military_flights()                  │
│     ├─ ADS-B NACp extracted per aircraft (line 790-796)        │
│     └─ GPS jamming zones computed inline (lines 1018-1063)     │
│        ├─ Snap coords to 1°×1° grid cells                      │
│        ├─ Count total vs. degraded aircraft (NACp < 8)         │
│        ├─ Filter: min 3 aircraft, ratio > 0.25                 │
│        └─ Store in latest_data["gps_jamming"]                  │
│                                                                  │
│  Slow-tier Scheduler (every 30min)                              │
│  └─ fetch_firms_fires()                                         │
│     ├─ GET NASA FIRMS CSV (NOAA-20 VIIRS, 24h global)         │
│     ├─ Parse: lat, lng, frp, brightness, confidence            │
│     ├─ Sort by FRP descending, keep top 5000                   │
│     └─ Store in latest_data["firms_fires"]                     │
│                                                                  │
│  API Endpoints                                                  │
│  ├─ GET /api/live-data/fast → gps_jamming                      │
│  └─ GET /api/live-data/slow → firms_fires                      │
│     (both: ETag-based 304 Not Modified caching)                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                             ↓ HTTP/JSON
┌─────────────────────────────────────────────────────────────────┐
│                   FRONTEND (Next.js/React)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Data Fetching (page.tsx)                                       │
│  ├─ fetchFastData → /api/live-data/fast (every 60s)            │
│  └─ fetchSlowData → /api/live-data/slow (every 120s)           │
│                                                                  │
│  GeoJSON Generation (MaplibreViewer.tsx)                        │
│  ├─ jammingGeoJSON → 1°×1° Polygon features (lines 390-422)   │
│  └─ firmsGeoJSON   → Point features w/ FRP icon (lines 467-493)│
│                                                                  │
│  Map Rendering                                                  │
│  ├─ Source: "gps-jamming" (GeoJSON polygons)                   │
│  │  ├─ Layer: gps-jamming-fill  (translucent red)              │
│  │  ├─ Layer: gps-jamming-outline (bright red edges)           │
│  │  └─ Layer: gps-jamming-label (GPS JAM XX%)                  │
│  │                                                              │
│  └─ Source: "firms-fires" (GeoJSON points, clustered)          │
│     ├─ Layer: firms-clusters (flame icon, 4 size steps)        │
│     └─ Layer: firms-viirs-layer (individual flame icons)       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## GPS Jamming Layer

### Acquisition (`backend/services/data_fetcher.py:1018-1063`)

- **Source**: ADS-B transponder data — NACp (Navigation Accuracy Category for Position) field
- **Primary provider**: adsb.lol (open-source ADS-B aggregator)
- **Update rate**: Computed inline every **60 seconds** during `fetch_flights()` / `fetch_military_flights()`
- NACp extracted per aircraft at line 790–796: `"nac_p": f.get("nac_p")`

**Detection algorithm:**
1. Snap each aircraft's position to a **1°×1° grid cell**: `grid_key = f"{int(lat)},{int(lng)}"`
2. Count total aircraft and degraded aircraft (`nac_p < 8`) per cell
3. Filter: minimum **3 aircraft** per cell (signal-to-noise floor)
4. Compute `ratio = degraded / total`
5. Discard cells with `ratio <= 0.25` (no meaningful jamming signal)
6. Assign severity:

| Ratio | Severity | Fill opacity |
|---|---|---|
| ≥ 0.75 | `"high"` | 0.45 |
| 0.50–0.75 | `"medium"` | 0.30 |
| 0.25–0.50 | `"low"` | 0.18 |

**Data structure:**
```json
{
  "lat": 51.5,
  "lng": 32.5,
  "severity": "high",
  "ratio": 0.82,
  "degraded": 9,
  "total": 11
}
```

### Rendering (`MaplibreViewer.tsx:1933-1974`)

- GeoJSON **Polygon** features — each a 1°×1° square (lines 390–422)
- Three Maplibre layers:
  1. `gps-jamming-fill` — translucent red fill (`#ff0040`), opacity from `severity`
  2. `gps-jamming-outline` — bright red edge, 1.5px, opacity 0.6
  3. `gps-jamming-label` — `"GPS JAM 82%"` text, size interpolated by zoom (8–12pt), red with black halo
- Toggle: `activeLayers.gps_jamming` — **ON by default**
- Served via `/api/live-data/fast`
- **Non-interactive**: no click popup; visualization-only

---

## NOAA-20 VIIRS Layer (NASA FIRMS)

### Acquisition (`backend/services/data_fetcher.py:1305-1344`)

- **Source**: NASA FIRMS — `https://firms.modaps.eosdis.nasa.gov/data/active_fire/noaa-20-viirs-c2/csv/J1_VIIRS_C2_Global_24h.csv`
- **Satellite**: NOAA-20 VIIRS (JPSS-1), 24-hour global thermal hotspot detection
- **No API key required** — public feed
- **Update rate**: Every **30 minutes** via `fetch_firms_fires()` in `update_slow_data()`

**Processing pipeline:**
1. Fetch raw CSV via `fetch_with_curl(url, timeout=30)`
2. Parse with `csv.DictReader`: extract `latitude`, `longitude`, `frp`, `bright_ti4`, `confidence`, `daynight`, `acq_date`, `acq_time`
3. Skip rows with parse errors (ValueError/TypeError)
4. Sort all rows by FRP descending
5. Retain **top 5,000** by FRP intensity

**FRP-based icon assignment:**

| FRP (MW) | Icon | Colors |
|---|---|---|
| ≥ 100 | `fire-darkred` | `#cc0000` → `#ff2200` |
| 20–99 | `fire-red` | `#ff2200` → `#ff8800` |
| 5–19 | `fire-orange` | `#ff8800` → `#ffcc00` |
| < 5 | `fire-yellow` | `#ffcc00` → `#fff5aa` |

**Data structure:**
```json
{
  "lat": -15.82,
  "lng": 28.44,
  "frp": 142.3,
  "brightness": 412.1,
  "confidence": "high",
  "daynight": "D",
  "acq_date": "2026-03-11",
  "acq_time": "0947"
}
```

### Rendering (`MaplibreViewer.tsx:1395-1438`)

- GeoJSON **Points** built in `useMemo` (lines 467–493)
- **Clustering enabled**: `clusterRadius=40`, `clusterMaxZoom=10`
- Updated via `useImperativeSource()` hook with **2000ms debounce** (avoids React reconciliation overhead for 5,000+ features)

**Flame SVG icons** generated by `makeFireSvg()` (lines 57–81):
- Four individual icons: `fire-yellow`, `fire-orange`, `fire-red`, `fire-darkred`
- Four cluster icons: `fire-cluster-sm/md/lg/xl` (32–56px)

**Two Maplibre layers:**
1. `firms-clusters` — cluster flame icons, stepped size at counts 10/50/200; shows abbreviated count text
2. `firms-viirs-layer` — individual flame icons; size interpolated 0.4×–1.0× by zoom level

- Toggle: `activeLayers.firms` — **OFF by default**
- Served via `/api/live-data/slow`
- **Non-interactive**: no click popup; visualization-only

---

## Scheduling & Polling

| | GPS Jamming | NOAA-20 VIIRS |
|---|---|---|
| **Backend tier** | Fast (60s) | Slow (30min) |
| **Frontend poll** | 60s | 120s |
| **API endpoint** | `/api/live-data/fast` | `/api/live-data/slow` |
| **ETag caching** | Yes | Yes |

---

## Quick Comparison

| | GPS Jamming | NOAA-20 VIIRS (FIRMS) |
|---|---|---|
| **Data source** | ADS-B NACp degradation | NASA FIRMS CSV |
| **Geometry** | Polygon (1°×1° grid) | Point (clustered) |
| **Max features** | ~1,000–2,000 cells | 5,000 hotspots |
| **Color scheme** | Red, opacity by severity | Yellow→Orange→Red by FRP |
| **Interactive** | No | No |
| **Default state** | ON | OFF |
| **Cross-layer ref** | Shares data with flights layer | Correlates with MODIS Terra imagery |

---

## Key Files

| Path | Purpose | Key Lines |
|---|---|---|
| `backend/services/data_fetcher.py:790-796` | NACp extraction from ADS-B | `"nac_p": f.get("nac_p")` |
| `backend/services/data_fetcher.py:1018-1063` | GPS jamming zone computation | Grid aggregation, severity thresholds |
| `backend/services/data_fetcher.py:1305-1344` | `fetch_firms_fires()` | CSV fetch, FRP sort, top-5000 cap |
| `backend/services/data_fetcher.py:2136-2172` | Scheduler (fast + slow tiers) | Job registration |
| `backend/main.py:80-108` | `/api/live-data/fast` | GPS jamming endpoint |
| `backend/main.py:110-142` | `/api/live-data/slow` | FIRMS endpoint |
| `frontend/src/components/MaplibreViewer.tsx:57-81` | `makeFireSvg()` | Flame SVG icon generator |
| `frontend/src/components/MaplibreViewer.tsx:238-255` | `useImperativeSource()` | Imperative GeoJSON update with debounce |
| `frontend/src/components/MaplibreViewer.tsx:390-422` | `jammingGeoJSON` useMemo | Polygon GeoJSON construction |
| `frontend/src/components/MaplibreViewer.tsx:467-493` | `firmsGeoJSON` useMemo | Point GeoJSON with icon assignment |
| `frontend/src/components/MaplibreViewer.tsx:1933-1974` | GPS jamming layers | Fill, outline, label layer config |
| `frontend/src/components/MaplibreViewer.tsx:1395-1438` | FIRMS layers | Cluster + individual icon layer config |
| `frontend/src/components/WorldviewLeftPanel.tsx:78-100` | Layer toggle UI | Radio/Flame icons, counts |
| `frontend/src/app/page.tsx:130-152` | Active layers state | Default `gps_jamming: true`, `firms: false` |
| `frontend/src/app/page.tsx:300-348` | Data polling | Fast (60s) + slow (120s) intervals with ETag |
