# Internet Outages & Data Centers — Technical Summary

> Codebase: `Shadowbroker-downstream` | Generated: 2026-03-11

---

## Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                      BACKEND (Python/FastAPI)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Scheduler (every 30 min)                                       │
│  ├─ fetch_internet_outages()                                    │
│  │  ├─ GET https://api.ioda.inetintel.cc.gatech.edu/v2/...     │
│  │  ├─ Filter: bgp|ping-slash24 only, severity >= 10%          │
│  │  ├─ Geocode: Nominatim OSM (cached)                         │
│  │  └─ Store in latest_data["internet_outages"]                │
│  │                                                              │
│  └─ fetch_datacenters()                                        │
│     ├─ Load from /backend/data/datacenters.json (7d cache)    │
│     ├─ If stale: GET GitHub (Ringmast4r/DC-Map)               │
│     ├─ Validate: Fix Southern Hemisphere signs, bbox check    │
│     └─ Store in latest_data["datacenters"]                    │
│                                                                  │
│  API Endpoints                                                 │
│  ├─ GET /api/live-data/slow → internet_outages + datacenters  │
│  ├─ GET /api/live-data → full payload                         │
│  └─ Caching: ETag-based 304 Not Modified                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                             ↓ HTTP/JSON
┌─────────────────────────────────────────────────────────────────┐
│                   FRONTEND (Next.js/React)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Data Fetching (page.tsx)                                      │
│  ├─ useEffect: fetch(`${API_BASE}/api/live-data/slow`)        │
│  ├─ Poll interval: ~5-10s                                     │
│  └─ Update state.data                                          │
│                                                                  │
│  Layer Toggle (WorldviewLeftPanel.tsx)                         │
│  ├─ activeLayers.internet_outages: boolean                   │
│  ├─ activeLayers.datacenters: boolean                        │
│  └─ Updated by user clicks                                    │
│                                                                  │
│  GeoJSON Generation (MaplibreViewer.tsx)                       │
│  ├─ useMemo(internetOutagesGeoJSON) → Features               │
│  ├─ useMemo(dataCentersGeoJSON) → Features                   │
│  └─ Rebuilt on data/layer changes                             │
│                                                                  │
│  Map Rendering                                                 │
│  ├─ Source: "internet-outages" (GeoJSON)                     │
│  │  ├─ Layer: internet-outages-pulse (outer ring)            │
│  │  ├─ Layer: internet-outages-layer (grey circle)           │
│  │  ├─ Layer: internet-outages-pct (severity %)              │
│  │  └─ Layer: internet-outages-label (region name)           │
│  │                                                             │
│  └─ Source: "datacenters" (GeoJSON, clustered)               │
│     ├─ Layer: datacenters-clusters (purple groups)           │
│     ├─ Layer: datacenters-cluster-count (count text)         │
│     └─ Layer: datacenters-layer (individual icons)           │
│                                                                 │
│  Interactive Popups (on click)                                │
│  ├─ Datacenter: name, company, city, country                │
│  └─ Outage cross-ref: Shows outages in same country         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Internet Outages

### Acquisition (`backend/services/data_fetcher.py:1413-1482`)

- **Source**: IODA (Georgia Tech) — `https://api.ioda.inetintel.cc.gatech.edu/v2/outages/alerts`
- **Update rate**: Every 30 minutes via `fetch_internet_outages()` in `update_slow_data()`
- **Query window**: Last 24 hours, up to 500 alerts

**Filtering logic:**
- Only accepts `bgp` and `ping-slash24` datasources (rejects `merit-nt` as unreliable)
- Only processes `etype == "region"` (not country-level)
- Drops alerts with severity < 10% (filters normal jitter)
- Severity formula: `(1 - value / historyValue) * 100`, clamped to [0, 100]
- Deduplicates per region — keeps the worst (highest severity) alert
- Caps output at 100 regions sorted by severity descending

**Geocoding** (`_geocode_region()`, lines 1391–1411):
- Resolves region+country names to lat/lng via **OpenStreetMap Nominatim**
- Results cached in-memory in `_region_geocode_cache`

**Data structure:**
```json
{
  "region_code": "NE",
  "region_name": "Nebraska",
  "country_code": "US",
  "country_name": "United States",
  "level": "outage",
  "datasource": "bgp",
  "severity": 45,
  "lat": 41.5,
  "lng": -99.8
}
```

### Rendering (`MaplibreViewer.tsx:2072-2134`)

- GeoJSON Points built per region in `useMemo` (lines 495–526)
- Four Maplibre layers stacked:
  1. `internet-outages-pulse` — outer semi-transparent grey ring, radius scales with severity (14–22px)
  2. `internet-outages-layer` — solid grey inner circle (`#888888`), radius 6–12px
  3. `internet-outages-pct` — symbol label showing `"45%"` (or `"!"` if 0)
  4. `internet-outages-label` — region name below the marker
- Toggle: `activeLayers.internet_outages` — **OFF by default**
- Served via `/api/live-data/slow`

---

## Data Centers

### Acquisition (`backend/services/data_fetcher.py:1556-1603`)

- **Source**: Curated GitHub dataset (`Ringmast4r/Data-Center-Map---Global`)
- **Local cache**: `/backend/data/datacenters.json`, valid for **7 days**; re-fetches from GitHub when stale

**Coordinate validation** (`_fix_dc_coords()`, lines 1529–1553):
- Fixes Southern Hemisphere sign errors for known countries (Australia, Brazil, Argentina, etc.) — if `lat > 0` for these, it negates it
- Validates against country bounding boxes; drops entries that still fail after sign-flip

**Data structure:**
```json
{
  "name": "AWS us-east-1a",
  "company": "Amazon Web Services",
  "city": "Virginia",
  "country": "United States",
  "lat": 37.25,
  "lng": -76.08
}
```

### Rendering (`MaplibreViewer.tsx:2136-2191`)

- GeoJSON Points built in `useMemo` (lines 528–545)
- **Clustering enabled**: `clusterRadius=30`, `clusterMaxZoom=8`
- Three Maplibre layers:
  1. `datacenters-clusters` — purple circles (`#7c3aed`), radius steps at counts 10/50
  2. `datacenters-cluster-count` — abbreviated count text in light purple
  3. `datacenters-layer` — individual SVG icons (server rack shape, `#a78bfa`)
- Toggle: `activeLayers.datacenters` — **OFF by default**
- Served via `/api/live-data/slow`

---

## Cross-Reference Feature

When clicking a data center, the popup (`MaplibreViewer.tsx:2412-2455`) searches `data.internet_outages` for any outage in the **same country**. If found, a red warning box lists affected regions and their severity percentages — linking infrastructure location to real-time connectivity events.

---

## Quick Comparison

| | Internet Outages | Data Centers |
|---|---|---|
| **Source** | IODA / Georgia Tech API | GitHub JSON (Ringmast4r) |
| **Update rate** | Every 30 min | 7-day local cache |
| **Geocoding** | OSM Nominatim (cached) | Pre-geocoded in dataset |
| **API endpoint** | `/api/live-data/slow` | `/api/live-data/slow` |
| **Map style** | Grey circles + severity labels | Clustered purple SVG icons |
| **Default state** | OFF | OFF |

---

## Key Files

| Path | Purpose | Key Functions |
|---|---|---|
| `backend/services/data_fetcher.py:1391-1603` | Data acquisition | `fetch_internet_outages()`, `fetch_datacenters()`, `_geocode_region()`, `_fix_dc_coords()` |
| `backend/main.py:80-142` | API endpoints | `/api/live-data/slow`, `/api/live-data` |
| `frontend/src/app/page.tsx:140-152` | Layer toggle state | Initial state (both OFF) |
| `frontend/src/components/MaplibreViewer.tsx:495-2191` | Map rendering | GeoJSON generation, layer styling, popup logic |
| `frontend/src/components/WorldviewLeftPanel.tsx:36-100` | Layer toggle UI | Layer list with icons and counts |
| `frontend/src/lib/api.ts:11-28` | API config | Base URL resolution |
