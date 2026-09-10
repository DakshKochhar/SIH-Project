# NammaFlood Backend — SIH26085

**Team FLOWCAST · Smart India Hackathon 2026 · Pilot city: Bengaluru**

A production-shaped prototype backend for street-level urban flood nowcasting
(0–3 hour horizon), fusing rainfall, terrain, land cover and stormwater
drainage data into explainable flood-risk, depth, and time-to-flood
predictions, plus dynamic road-risk scoring and flood-aware routing.

> **DEMO_MODE is on by default.** Every rainfall/drainage/terrain/land-cover
> value is clearly-labelled **simulated** data (`is_simulated: true`,
> `-DEMO`/`-SIM` source suffixes), generated deterministically from
> `DEMO_RANDOM_SEED` around real Bengaluru flood-prone locations. Nothing in
> this codebase fabricates or claims to be real IMD/KSNDMC/BBMP/KSRSAC/NRSC
> data. See [Connecting real data sources](#connecting-real-data-sources).

---

## 1. Architecture

```
Rainfall/Nowcast    Terrain DEM    Land Cover        Drain Network
(IMD/KSNDMC)        (CartoDEM)     (ESA WorldCover)   (BBMP/KSRSAC)
      |                  |               |                  |
      +---------- app/ingestion/* (provider interfaces) -----+
                              |
                    app/services/rainfall_service.py  (fusion, IDW)
                    app/services/runoff_service.py     (catchment runoff)
                    app/services/drainage_service.py   (loading, bottlenecks)
                    app/geospatial/terrain.py          (low-point / terrain risk)
                              |
                    app/ml/model.py  --  Hybrid Physics + ML ensemble
                              |
              +---------------+----------------+
              |               |                |
        flood_probability  depth_cm    time_to_flood_min
              |               |                |
              +-------- app/services/flood_prediction_service.py
                              |
                    app/services/road_risk_service.py  (0-100 road risk)
                    app/services/routing_service.py    (flood-aware routing, NetworkX)
                    app/services/alert_service.py       (threshold alerts)
                              |
                        FastAPI REST API (app/api/routes/*)
                              |
                    Frontend / Mobile / Control Room dashboard
```

Every threshold (drain-loading bands, time-to-flood bands, road-risk bands,
alert thresholds, runoff coefficients, hybrid-engine weights) lives in
`app/core/config.py`, not hard-coded in business logic.

## 2. Project layout

```
nammaflood-backend/
├── app/
│   ├── main.py                 # FastAPI app, CORS, rate limiting, startup
│   ├── api/routes/             # rainfall, flood, drainage, roads, routes,
│   │                           #   dashboard, auth, model, alerts, health
│   ├── core/                   # config (all thresholds), security (JWT), logging
│   ├── db/                     # SQLAlchemy base + session (Postgres/PostGIS,
│   │                           #   degrades to SQLite+WKT for local dev/tests)
│   ├── models/                 # ORM models: rainfall, drainage, terrain,
│   │                           #   flood, road, user
│   ├── schemas/                # Pydantic request/response + GeoJSON models
│   ├── services/                # rainfall fusion, runoff, drainage loading,
│   │                           #   flood prediction, road risk, routing, alerts
│   ├── ingestion/               # provider interfaces + DEMO implementations
│   │                           #   for rainfall/drainage/terrain/landcover/
│   │                           #   historical-flood-events
│   ├── ml/                     # feature vector, hybrid model, training, predict
│   ├── geospatial/              # haversine/IDW, terrain risk, drainage graph,
│   │                           #   GeoJSON builders
│   ├── demo/                   # deterministic Bengaluru demo dataset generator
│   └── workers/                # asyncio background update loop
├── tests/                      # pytest unit + integration tests (59 tests)
├── scripts/seed_db.py          # standalone DB seed script
├── alembic/                    # migration environment (autogenerate-ready)
├── Dockerfile
├── docker-compose.yml           # backend + postgis + redis
├── requirements.txt
├── .env.example
└── alembic.ini
```

## 3. Running locally (no Docker)

Requires Python 3.11+. `geopandas`/`rasterio` need GDAL/GEOS/PROJ system
libraries — if you don't have them and only want the API (which uses its own
lightweight geospatial helpers, not GeoPandas, for the demo pipeline), you
can skip those two lines in `requirements.txt`.

```bash
cd nammaflood-backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

cp .env.example .env
# For local dev without Postgres, edit .env and set:
#   DATABASE_URL=sqlite:///./nammaflood.db

uvicorn app.main:app --reload --port 8000
```

On startup the app auto-creates tables and seeds three demo accounts
(see [Authentication](#5-authentication)). Swagger docs: `http://localhost:8000/docs`.

### Running tests

```bash
pip install -r requirements.txt
pytest tests/ -q
# 59 passed
```

Tests run against an isolated SQLite DB (`tests/conftest.py` sets
`DATABASE_URL` before the app is imported) — no Postgres required.

## 4. Running with Docker

```bash
cd nammaflood-backend
cp .env.example .env
docker compose up --build
```

This starts `postgis` (PostgreSQL 16 + PostGIS), `redis`, and the `backend`
API on `http://localhost:8000`. Tables are created automatically on
startup; run Alembic if you'd rather manage schema via migrations:

```bash
docker compose exec backend alembic revision --autogenerate -m "init"
docker compose exec backend alembic upgrade head
```

## 5. Authentication

JWT bearer auth with three roles: `CITIZEN`, `MUNICIPAL_OPERATOR`, `ADMIN`.
Demo accounts (seeded automatically, password `ChangeMe123!` — **change
before any real deployment**):

| Email | Role |
|---|---|
| `admin@nammaflood.demo` | ADMIN |
| `operator@nammaflood.demo` | MUNICIPAL_OPERATOR |
| `citizen@nammaflood.demo` | CITIZEN |

```bash
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "citizen@nammaflood.demo", "password": "ChangeMe123!"}'
# => {"access_token": "...", "token_type": "bearer", "role": "CITIZEN"}
```

Use the token as `Authorization: Bearer <token>` on subsequent calls.

Role access:
- **CITIZEN**: flood risk, hotspots, map, roads, safe routes, alerts, rainfall.
- **MUNICIPAL_OPERATOR**: + drainage status/bottlenecks, prediction detail.
- **ADMIN**: + `/model/retrain`.

## 6. Sample API requests

```bash
TOKEN="<paste access_token here>"
BASE=http://localhost:8000/api/v1

# Health (no auth)
curl http://localhost:8000/health

# Current rainfall
curl -H "Authorization: Bearer $TOKEN" "$BASE/rainfall/current"

# Rainfall forecast (0-3hr)
curl -H "Authorization: Bearer $TOKEN" "$BASE/rainfall/forecast?horizon_hours=3"

# Flood prediction at a point (Silk Board Junction)
curl -H "Authorization: Bearer $TOKEN" \
  "$BASE/flood/predict?latitude=12.9172&longitude=77.6228&horizon=180"

# Area flood map (GeoJSON)
curl -H "Authorization: Bearer $TOKEN" "$BASE/flood/map"

# Flood hotspots
curl -H "Authorization: Bearer $TOKEN" "$BASE/flood/hotspots?min_risk=HIGH"

# Drainage status (operator+)
curl -H "Authorization: Bearer $OPERATOR_TOKEN" "$BASE/drainage/status"

# Drainage bottlenecks
curl -H "Authorization: Bearer $TOKEN" "$BASE/drainage/bottlenecks"

# Road risk
curl -H "Authorization: Bearer $TOKEN" "$BASE/roads/risk"

# Safe route
curl -H "Authorization: Bearer $TOKEN" \
  "$BASE/routes/safe?source_lat=12.9172&source_lng=77.6228&destination_lat=13.0358&destination_lng=77.5970"

# Dashboard summary
curl -H "Authorization: Bearer $TOKEN" "$BASE/dashboard/summary"

# Active alerts
curl -H "Authorization: Bearer $TOKEN" "$BASE/alerts/active"

# Retrain model (admin only — will 422 until enough historical data exists)
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" "$BASE/model/retrain"
```

### Example: `GET /api/v1/flood/predict`

```json
{
  "prediction_id": "f84a0a93-56f7-40b1-bdc6-f5f42bafcabc",
  "latitude": 12.9172,
  "longitude": 77.6228,
  "timestamp": "2026-09-10T12:36:58.156553+00:00",
  "forecast_horizon_min": 180,
  "flood_probability": 0.91,
  "risk_level": "CRITICAL",
  "depth_cm": 40.5,
  "time_to_flood_min": 5,
  "confidence": 0.65,
  "data_quality": "DEMO",
  "main_drivers": [
    "High rainfall intensity",
    "Drainage segment approaching or over capacity",
    "Low terrain elevation / local low point",
    "High impervious surface"
  ],
  "explanation": {
    "rainfall_contribution": 0.35,
    "terrain_contribution": 0.155,
    "imperviousness_contribution": 0.122,
    "drainage_loading_contribution": 0.283
  },
  "engine": "hybrid_rules",
  "is_estimated": true,
  "is_simulated": true
}
```

`confidence` and `data_quality` are always present so a frontend can visibly
flag low-confidence/DEMO predictions rather than presenting them as
authoritative.

## 7. Hybrid physics + ML engine

```
INPUT FEATURES -> [Physics/Rule Model] + [ML Model, if trained] -> Ensemble -> Final Risk
```

- The **rule-based component** (`app/ml/model.py::RuleBasedComponent`) is
  always available: a transparent weighted combination of rainfall,
  terrain, imperviousness and drainage-loading signals
  (`RULE_WEIGHT_*` in config).
- The **ML component** activates only once `POST /api/v1/model/retrain`
  succeeds (requires ≥ `MODEL_MIN_TRAINING_ROWS` historical events — the
  bundled demo historical dataset is intentionally smaller than that, so a
  fresh checkout honestly reports `engine: "hybrid_rules"` and a 422 on
  retrain until a real/larger historical-event feed is connected).
- When both are available, the ensemble is a weighted blend and
  `confidence` is bumped slightly to reflect the extra signal.

## 8. Connecting real data sources

Every ingestion source has a `Demo*Provider` and a real-provider placeholder
class that raises `NotImplementedError` (see `app/ingestion/*.py`). To go
live:

1. Implement the real provider class against the live API (IMD/KSNDMC,
   BBMP/KSRSAC, NRSC/Bhuvan CartoDEM, ESA WorldCover), returning data in the
   same shape as the demo provider.
2. Set the matching `*_PROVIDER` env var away from `"demo"`.
3. Add credentials to `.env` (`IMD_API_KEY`, `KSNDMC_API_KEY`, `BHUVAN_API_KEY`).

No other code changes are required — services depend on the abstract
interfaces in `app/ingestion/base.py`, not on the demo classes.

## 9. Known limitations (stated deliberately, not hidden)

- Pipe-level hydraulic attributes and live blockage status are not fully
  open in Bengaluru's public GIS layers — the drainage model works from
  segment-level capacity and simulated loading until such a feed exists.
- Feeds can have gaps or different update cycles; `data_quality` on every
  prediction (`FULL`/`PARTIAL`/`DEMO`) reflects this.
- CartoDEM (~30m) / ESA WorldCover (~10m) resolutions do not support
  centimetre-level terrain precision — depth/probability are always
  labelled `is_estimated: true`, never presented as observed measurements.
- Pure ML can be brittle on unseen extremes — hence the hybrid architecture
  with a rules-only fallback that never depends on a trained model existing.

## 10. Roadmap alignment (build phases)

| Phase | Status |
|---|---|
| 1. FastAPI + PostGIS models + health | ✅ |
| 2. Demo rainfall/drainage/terrain/land-cover | ✅ |
| 3. Runoff + drainage loading | ✅ |
| 4. Hybrid flood prediction engine | ✅ |
| 5. Depth + time-to-flood + explainability | ✅ |
| 6. Road risk + safe routing | ✅ |
| 7. Alerts + dashboard | ✅ |
| 8. Auth + Docker + tests + docs | ✅ |
