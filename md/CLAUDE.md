# RigVision-3D Project Context

## PROJECT IDENTITY

**Name:** RigVision-3D
**What it is:** A real-time 3D digital twin monitoring system for ONGC (Oil and Natural Gas Corporation) drilling rigs. It fuses a CAD-derived 3D model with live multi-camera video feeds and IoT sensor data into a single browser-based interactive dashboard.
**Team:** 4 B.Tech students from LNMIIT, Jaipur (Communication and Computer Engineering dept), doing an 8-week summer internship at ONGC (May–July 2026).
**Hardware:** 4 phone cameras (DroidCam RTSP, 2 per room for stereo triangulation), RTX 4070 GPU, procedural 3D model (2 stacked rooms, no corridor).

---

## WHAT THE SYSTEM DOES (5 PILLARS)

### Pillar 1: 3D Digital Twin
- Browser-based 3D rendering using Three.js + React (@react-three/fiber)
- Procedural model: 2 stacked rooms — Room A (floor 0) and Room B (floor 1) directly above it, no corridor (defined in `cad/zone_definitions.json`)
- Zones are translucent bounding boxes that change color: green=normal, amber=warning, red=critical
- Equipment is clickable with metadata popups
- Camera modes: OrbitControls (default), PointerLockControls (walkthrough), MapControls (top-down)

### Pillar 2: Multi-Camera CV Pipeline
- 4 phone cameras stream via DroidCam RTSP (2 per room, overlapping FOV)
- YOLOv8l detects persons + PPE (hard hat, vest, goggles) in one pass on RTX 4070
- BoT-SORT tracks persons per-camera with persistent IDs + ReID embeddings
- **Per-zone-group processing:** each room's 2 cameras are fused independently — cross-camera matching (ArUco identity + epipolar geometry) then DLT triangulation → 3D room coordinates. A person's zone is decided by *which camera group saw them*, so no global world frame is needed and rooms can't cross-contaminate.
- **Single in-process pipeline** (`cv/pipeline.py`): detection → tracking → cross-camera → triangulation → sensor fusion all run in one process and write Redis directly. No Kafka in the CV path (the old `ccm-matches`/`3d-locations` topics and their services were removed as redundant).
- Output: JSON array of tracked persons written to Redis at ~10Hz

### Pillar 3: Sensor Ingestion (the "seam")
- Sensor types: temperature, vibration (g RMS), noise (dB), gas (H₂S ppm), pressure. Each zone has **exactly 5 sensors (one per type)**, defined in `cad/zone_definitions.json` with per-sensor `warning`/`critical` thresholds.
- **Thresholds are bidirectional:** pressure sensors also carry `warning_low`/`critical_low` (loss of pressure is as dangerous as overpressure). `anomaly_evaluator` reports `breach_direction` (`high`/`low`) in `threshold_context`; a low breach maps to the `<type>_low` KG symptom (e.g. `pressure_low`) so it routes to loss-of-pressure failure modes.
- **The seam:** a single Redis key `rigvision:sensors:latest`, keyed by `sensor_id`. Any producer writes it; the CV pipeline only ever reads it. Swap the producer, nothing downstream changes.
  - **Now:** a manual **Sensor Console** (React, its own dev port `:5174`) with one slider per sensor. Edits are local; **"SEND TO REDIS"** commits (hash-gated — only sends if values changed). `source:"manual"` readings are **never stale** (persist until changed).
  - **Future:** a small MQTT→Redis bridge writes the same key with `source:"mqtt"`; those readings expire after `SENSOR_STALE_SECONDS` (offline detection). No pipeline change required.
- The CV pipeline fuses current sensor readings + person occupancy/PPE into `rigvision:zones` (`build_zone_states`): per-sensor threshold check → zone status, worst-case aggregation per type, missing/stale → `null` ("NO DATA").

### Pillar 4: ~~Compliance Engine~~ (removed)
- The always-on YAML rule engine and `rigvision:violations:latest` were **removed**. Safety reporting is now on-demand LLM diagnostics (Pillar 5). PPE/occupancy violations will be re-added when PPE detection is integrated (`zone.ppe_violations` is already carried in the zone state, currently empty).

### Pillar 5: Knowledge Graph + On-Demand LLM Diagnostics
- Neo4j graph (seeded by `knowledge/graph/seed_graph.py` from `cad/zone_definitions.json` + `knowledge/thresholds/threshold_registry.json`): Zone (`room_1`/`room_2`), Device (rig equipment: mud pump, control panel, compressor, wellhead), Sensor, Manual, ThresholdSpec, FailureMode, Symptom, Action nodes.
- **Manual-derived thresholds:** limits come from device manuals, not hardcoded JSON. Offline: `knowledge/extraction/manual_threshold_extractor.py` (local LLM) extracts candidate ThresholdSpecs from `knowledge/documents/ONGC_Device_Manuals.txt` → human validates → `threshold_registry.json` → seeded into Neo4j. Runtime: `backend/services/threshold_resolver.py` resolves per-sensor limits with priority **device manual → zone environmental (HSE) → zone_definitions.json fallback** (also the safety net when Neo4j is down). Both `warning_low`/`critical_low` are carried by the resolver. Deterministic comparison in `backend/services/anomaly_evaluator.py` — the LLM never decides thresholds live. Inspect via `GET /api/thresholds`; re-resolve via `POST /api/thresholds/refresh` after re-seeding.
- **Trigger:** "RUN DIAGNOSTICS" button in the Sensor Console → `POST /api/diagnostics/run`. Backend threshold-checks **every zone** against current sensor data (using resolved thresholds) and publishes **one Kafka alert per flagged zone** (`rigvision_alerts`), each carrying a `threshold_context` explaining which device/manual limit fired plus `breach_direction`. No flags → instant `all_clear`, no LLM call.
- **Pipeline:** `anomaly_listener` (`knowledge/extraction/anomaly_listener.py`) consumes each alert → emits per-stage progress to `rigvision:diag:progress` → KG Cypher query → ChromaDB vector search → LLM prompt → root-cause JSON → Kafka `rigvision_diagnostics` → backend consumer → Redis `rigvision:diagnostics` → WebSocket → frontend.
- **Live progress stages** (emitted to `rigvision:diag:progress` hash, keyed by `event_id`): `generating_query` → `getting_subgraph` → `subgraph_ready` → `getting_chunks` → `chunks_ready` → `writing_answer` → `done` / `error`. Entries expire after 600s; WS bridge prunes entries older than 120s from UI state.
- **Anti-hallucination:** the LLM schema has `anomaly_detected`; the prompt forces "No issue detected" when `triggered_sensors` is empty / data is within limits.
- **Models:** **Both embeddings and generation run locally** via LM Studio (OpenAI-compatible REST, called with `requests` — no `openai`/`google` SDK). Two models loaded simultaneously in LM Studio (`n_parallel=4`):
  - Generation: `qwen2.5-7b-instruct-1m` configured by `LLM_BASE_URL`/`LLM_API_KEY`/`LLM_MODEL`
  - Embeddings: `text-embedding-nomic-embed-text-v1.5` configured by `EMBED_BASE_URL`/`EMBED_API_KEY`/`EMBED_MODEL` (shared helper `knowledge/agent_layer/embeddings.py`)
  - Changing `EMBED_MODEL` requires re-running `rag_ingestion.py` (ChromaDB stores one vector dimensionality). Zone→KG mapping: `zone_a→room_1`, `zone_b→room_2`.
- **API auth:** set `RIGVISION_API_KEY` (+ `VITE_API_KEY` in the frontend) to require `X-API-Key` on mutating endpoints; `RUN DIAGNOSTICS` is rate-limited by `DIAGNOSTICS_MIN_INTERVAL`. Empty key = auth off (dev).
- **Dedup:** backend `_last_alert_signatures` prevents re-alerting the same zone/severity/sensors combo.
- **AI Diagnostics Hub:** the full diagnostics UI (`DiagnosticsLive.jsx`) lives in a separate browser tab at `/diagnostics` or `/diagnostics/:eventId`. It shows a left alert-log list + right detail pane. In-flight events show a staged live pipeline timeline; completed events show a full report with telemetry grid (▼/▲ breach direction), threshold context, Neo4j subgraph, reasoning, and recommended action. Opened from the notification toast or "View Reports" in TopBar.

---

## TECH STACK

| Layer | Tech |
|-------|------|
| 3D Rendering | Three.js, @react-three/fiber, @react-three/drei |
| UI Framework | React 18, Zustand, Vite |
| Backend API | FastAPI, Uvicorn |
| CV Detection | YOLOv8l (Ultralytics) |
| CV Tracking | BoT-SORT (via boxmot), IoU fallback |
| Video Ingest | OpenCV, FFmpeg |
| Sensor Transport | Manual Sensor Console now (Redis seam); MQTT/EMQX later |
| Alert Bus | Kafka (`rigvision_alerts`, `rigvision_diagnostics`) |
| Database | PostgreSQL 16 + TimescaleDB |
| Cache | Redis 7 |
| Knowledge Graph | Neo4j 5.x |
| Vector Store | ChromaDB (port 8100) |
| LLM (generation) | `qwen2.5-7b-instruct-1m` via LM Studio (local, OpenAI-compatible REST) |
| LLM (embeddings) | `text-embedding-nomic-embed-text-v1.5` via LM Studio (local, same server) |

---

## ENVIRONMENT VARIABLES (`.env` — never commit)

```
# Answer-generation LLM (LM Studio)
LLM_BASE_URL=http://localhost:1234/v1
LLM_API_KEY=lm-studio
LLM_MODEL=qwen2.5-7b-instruct-1m

# Embeddings LLM (LM Studio, same server, second model slot)
EMBED_BASE_URL=http://localhost:1234/v1
EMBED_API_KEY=lm-studio
EMBED_MODEL=text-embedding-nomic-embed-text-v1.5

# API auth (empty = auth off in dev)
RIGVISION_API_KEY=
VITE_API_KEY=

# Diagnostics rate-limiting
DIAGNOSTICS_MIN_INTERVAL=30   # seconds between runs

# Sensor staleness (for future MQTT source)
SENSOR_STALE_SECONDS=30

# Anomaly listener concurrency
LISTENER_WORKERS=3
```

Note: `.env` also has a `GEMINI_API_KEY` (used only if Gemini embeddings are re-enabled; currently unused at runtime — embeddings are local). **Rotate this key** — it was previously hardcoded in source and may be in git history.

---

## REDIS DATA CONTRACTS

### `rigvision:persons` (CV pipeline → Redis at ~10Hz)
```json
[{"id": 1, "x": 3.2, "y": 0.05, "z": 2.5, "zone": "zone_a", "posture": "standing", "ppe": {"hardhat": true, "vest": false, "goggles": false}, "confidence": 0.91, "cameras_visible": 1}]
```

### `rigvision:zones` (CV pipeline `build_zone_states` → Redis)
```json
{"zone_a": {"status": "normal", "label": "Room A", "floor": 0, "sensor_types": ["gas_h2s","noise","pressure","temperature","vibration"], "temperature": 28.3, "vibration": 1.2, "noise": 72, "gas_h2s": 0.5, "pressure": 12.0, "person_count": 2, "ppe_violations": [], "updated_at": 1716969600}}
```
Sensor values are `null` when that sensor is missing/stale (renders "NO DATA").

### `rigvision:sensors:latest` (THE SEAM — Sensor Console / future MQTT → Redis)
```json
{"temp_a": {"value": 28.3, "updated_at": 1716969600, "source": "manual"}}
```
Keyed by `sensor_id` (from `zone_definitions.json`). `source:"manual"` never expires; `source:"mqtt"` expires after `SENSOR_STALE_SECONDS`.

### `rigvision:diagnostics` (anomaly_listener via Kafka → backend → Redis)
```json
[{"event_id": "anom_1716969600_zone_b", "zone_id": "zone_b", "severity": "CRITICAL", "anomaly_detected": true, "primary_diagnosis": "Motor Burnout", "confidence_score": 85, "reasoning": "...", "recommended_action": "...", "triggered_sensors": ["temperature"], "telemetry_snapshot": {"temperature": 68.0}, "threshold_context": {"temperature": {"warning": 55, "critical": 65, "breach_direction": "high"}}, "timestamp": 1716969600000}]
```
Stored newest-first (`.insert(0, diag)` in backend). `anomaly_detected: false` entries are valid (all-clear after LLM gate check).

### `rigvision:diag:progress` (anomaly_listener → Redis hash, keyed by event_id)
```json
{
  "anom_1716969600_zone_b": "{\"event_id\": \"anom_...\", \"zone_id\": \"zone_b\", \"stage\": \"writing_answer\", \"subgraph\": \"...\", \"chunks\": \"...\", \"updated_at\": 1716969600000}"
}
```
Hash expires after 600s. Backend WS bridge reads `hgetall` each tick and prunes entries older than 120s before broadcasting as `diag_progress` in `realtime_update`.

---

## RUN ORDER (local dev)

Infra first: **Redis**, **Kafka**, **Neo4j**, **ChromaDB** (port 8100), and **LM Studio** (Developer → Start Server; load **both** `qwen2.5-7b-instruct-1m` and `text-embedding-nomic-embed-text-v1.5`, set `n_parallel=4`).

1. **Backend API** — `python backend/main.py` (FastAPI on :8000; REST + WebSocket, Kafka producer for `/api/diagnostics/run`, diagnostics consumer).
2. **CV pipeline** — `python cv/pipeline.py --mode demo` (fake people, real sensor feed; or `--mode live/video`). Writes `rigvision:persons` + `rigvision:zones`.
3. **Diagnostics listener** — `python knowledge/extraction/anomaly_listener.py` (Kafka → KG → ChromaDB → LM Studio → `rigvision_diagnostics`; emits per-stage progress to `rigvision:diag:progress`).
4. **Frontend (dashboard)** — `cd frontend && npm run dev` → `:5173`.
5. **Frontend (sensor console)** — `cd frontend && npm run dev:sensors` → `:5174`.

**Flow:** Sensor Console sliders → SEND TO REDIS → RUN DIAGNOSTICS → per-zone Kafka alerts → anomaly_listener emits live progress → LLM diagnoses → AI Diagnostics Hub in separate tab.

**One-time setup after model/seed changes:**
- Re-seed Neo4j: `python knowledge/graph/seed_graph.py`
- Re-ingest RAG: `python knowledge/agent_layer/rag_ingestion.py` (required whenever `EMBED_MODEL` changes — Chroma vector dimensionality is fixed per collection)
- Refresh thresholds: `Invoke-RestMethod -Method Post -Uri http://localhost:8000/api/thresholds/refresh`

---

## KEY FILES & ARCHITECTURE

### Backend
| File | Purpose |
|------|---------|
| `backend/main.py` | FastAPI app; WS bridge reads `rigvision:diag:progress` hash + broadcasts `diag_progress`; auth middleware; rate-limit on `/api/diagnostics/run` |
| `backend/services/anomaly_evaluator.py` | Bidirectional threshold check; sets `breach_direction: high/low` in `threshold_context` |
| `backend/services/threshold_resolver.py` | Resolves per-sensor limits (device manual → HSE → fallback); carries `warning_low`/`critical_low` |

### Knowledge / Diagnostics
| File | Purpose |
|------|---------|
| `knowledge/extraction/anomaly_listener.py` | Kafka consumer; lazy singletons; ThreadPoolExecutor (`LISTENER_WORKERS=3`); emits per-stage progress to Redis; topology cache |
| `knowledge/extraction/graph_extractor.py` | Neo4j subgraph queries; `_topology_cache` for static inter-equipment queries (computed once per driver lifetime) |
| `knowledge/extraction/query_generator.py` | Cypher query generation; low-breach → `pressure_low` symptom mapping |
| `knowledge/agent_layer/diagnostic_agent.py` | LLM diagnostic agent; generation via LM Studio REST (`requests`); `retrieve_manuals()` + `generate_answer()` split for stage-by-stage progress |
| `knowledge/agent_layer/embeddings.py` | Shared embed helper; reads `EMBED_BASE_URL`/`EMBED_API_KEY`/`EMBED_MODEL` |
| `knowledge/agent_layer/rag_ingestion.py` | Ingests `ONGC_Device_Manuals.txt` into ChromaDB using local embeddings; re-run when EMBED_MODEL changes |
| `knowledge/graph/seed_graph.py` | Seeds Neo4j; includes `pressure_low` symptom + 6 low-pressure failure modes (3 device-level, 3 zone-level) |
| `knowledge/thresholds/threshold_registry.json` | Validated ThresholdSpecs; pressure entries include `warning_low`/`critical_low` |
| `knowledge/documents/ONGC_Device_Manuals.txt` | Device manuals with loss-of-pressure failure modes added |
| `cad/zone_definitions.json` | Zone/sensor definitions; pressure sensors have `warning_low: 4, critical_low: 2` |

### Frontend
| File | Purpose |
|------|---------|
| `frontend/src/main.jsx` | Entry point; imports `authHandoff.js` first |
| `frontend/src/authHandoff.js` | Seeds sessionStorage from `window.opener` before React boots (auth handoff for new tabs) |
| `frontend/src/components/AppRouter.jsx` | Routes: `/` → App, `/diagnostics` + `/diagnostics/:eventId` → DiagnosticsLive, `/documents/manuals` → ManualsViewer |
| `frontend/src/components/DiagnosticsLive.jsx` | Full AI Diagnostics Hub (separate tab); alert log list + detail pane; `LivePipeline` staged timeline for in-flight events; `ReportDetail` for completed; unified `events` merges diagnostics + diagProgress |
| `frontend/src/components/NotificationAlert.jsx` | Anomaly toast; solid `var(--bg-panel)` background (no `backdrop-filter: blur` — causes WebGL jank); `openLive()` anchors to exact `event_id`; refresh/stale guards |
| `frontend/src/components/TopBar.jsx` | "View Reports" → `window.open('/diagnostics', '_blank')` (no `noopener` — required for auth handoff) |
| `frontend/src/stores/useRigStore.js` | Zustand store; `diagProgress: {}` state fed from WS `data.diag_progress` |
| `frontend/src/App.jsx` | Main dashboard; `DiagnosticsModal` removed (hub is now in separate window) |

### Deleted / Dead Files
- `frontend/src/components/DiagnosticsModal.jsx` — no longer used; can be deleted when confirmed.

---

## ZONE LAYOUT

Two stacked rooms (no corridor). Room B sits directly above Room A. Each room is an
8×6m bay, 3.4m tall; each is covered by 2 overlapping cameras.

```
   ┌────────────┐
   │   ROOM B   │   floor 1  (zone_b)  cam2 + cam3   Y: 3.4──6.8
   │  (zone_b)  │
   ├────────────┤
   │   ROOM A   │   floor 0  (zone_a)  cam0 + cam1   Y: 0────3.4
   │  (zone_a)  │
   └────────────┘
   X: 0────8   Z: 0────6
```

Total: 8m (X) × 6m (Z) × 6.8m (Y, two 3.4m floors). Origin at the ground-floor corner of
Room A. Y = up. Camera groups: `zone_a → [cam0, cam1]`, `zone_b → [cam2, cam3]`. The 3D
model is hand-built (procedural) in `frontend/src/components/Scene3D.jsx` — no external
GLTF asset.

---

## KNOWN ISSUES / PENDING

1. **`sensorsBreached()` in `NotificationAlert.jsx` only checks upper bounds** (`val >= meta.warning/critical`). Low-side pressure breaches (`val <= meta.warning_low`) are not detected, causing signature mismatch — the toast shows a stale old diagnosis instead of the new low-pressure one. Fix: add `|| (meta.warning_low != null && val <= meta.warning_low) || (meta.critical_low != null && val <= meta.critical_low)` to the check. Also verify `zone.sensor_meta` carries `warning_low`/`critical_low` from the backend `build_zone_states` output.
2. **`DiagnosticsModal.jsx`** is a dead file — safe to delete.
3. **GPU contention** on RTX 4070: YOLO + Qwen generation + nomic embeddings all share one GPU. Generation latency is inherent; tunable via `max_tokens` cap in `diagnostic_agent.py` or `LISTENER_WORKERS` concurrency.

---

## CONVENTIONS

- Python 3.11+, type hints encouraged
- Functional React components, hooks only
- Zustand for state, Three.js via @react-three/fiber
- Redis keys prefixed with `rigvision:`
- Coordinate system: X = length, Y = up, Z = width. Units = meters.
- **Never use `openai` or `google` SDK** — all LLM calls use `requests` (OpenAI-compatible REST)
- **Never commit `.env`** — it contains secrets
- `window.open` for the diagnostics tab must **NOT** include `'noopener'` — the new tab reads auth via `window.opener.sessionStorage` in `authHandoff.js`
