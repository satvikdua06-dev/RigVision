# RigVision-3D Project Context

## PROJECT IDENTITY

**Name:** RigVision-3D
**What it is:** A real-time 3D digital twin monitoring system for ONGC (Oil and Natural Gas Corporation) drilling rigs. It fuses a CAD-derived 3D model with live multi-camera video feeds and IoT sensor data into a single browser-based interactive dashboard.
**Team:** 4 B.Tech students from LNMIIT, Jaipur (Communication and Computer Engineering dept), doing an 8-week summer internship at ONGC (May–July 2026).
**Hardware:** 3 phone cameras (DroidCam RTSP), RTX 4070 GPU, procedural 3D model (2 rooms + corridor).

---

## WHAT THE SYSTEM DOES (5 PILLARS)

### Pillar 1: 3D Digital Twin
- Browser-based 3D rendering using Three.js + React (@react-three/fiber)
- Procedural model: 2 rooms + 1 corridor (defined in `cad/zone_definitions.json`)
- Zones are translucent bounding boxes that change color: green=normal, amber=warning, red=critical
- Equipment is clickable with metadata popups
- Camera modes: OrbitControls (default), PointerLockControls (walkthrough), MapControls (top-down)

### Pillar 2: Multi-Camera CV Pipeline
- 3 phone cameras stream via DroidCam RTSP
- YOLOv8l detects persons + PPE (hard hat, vest, goggles) in one pass on RTX 4070
- BoT-SORT tracks persons per-camera with persistent IDs + ReID embeddings
- Cross-camera matching uses ArUco identity and epipolar geometry
- DLT triangulation converts 2D pixel pairs → 3D room coordinates
- Output: JSON array of tracked persons written to Redis at ~10Hz

### Pillar 3: Sensor Ingestion (the "seam")
- Sensor types: temperature, vibration (g RMS), noise (dB), gas (H₂S ppm), pressure. Each zone has **exactly 5 sensors (one per type)**, defined in `cad/zone_definitions.json` with per-sensor `warning`/`critical` thresholds.
- **The seam:** a single Redis key `rigvision:sensors:latest`, keyed by `sensor_id`. Any producer writes it; the CV pipeline only ever reads it. Swap the producer, nothing downstream changes.
  - **Now:** a manual **Sensor Console** (React, its own dev port `:5174`) with one slider per sensor. Edits are local; **"SEND TO REDIS"** commits (hash-gated — only sends if values changed). `source:"manual"` readings are **never stale** (persist until changed).
  - **Future:** a small MQTT→Redis bridge writes the same key with `source:"mqtt"`; those readings expire after `SENSOR_STALE_SECONDS` (offline detection). No pipeline change required.
- The CV pipeline fuses current sensor readings + person occupancy/PPE into `rigvision:zones` (`build_zone_states`): per-sensor threshold check → zone status, worst-case aggregation per type, missing/stale → `null` ("NO DATA").

### Pillar 4: ~~Compliance Engine~~ (removed)
- The always-on YAML rule engine and `rigvision:violations:latest` were **removed**. Safety reporting is now on-demand LLM diagnostics (Pillar 5). PPE/occupancy violations will be re-added when PPE detection is integrated (`zone.ppe_violations` is already carried in the zone state, currently empty).

### Pillar 5: Knowledge Graph + On-Demand LLM Diagnostics
- Neo4j graph (seeded by `knowledge/graph/seed_graph.py` from `cad/zone_definitions.json` + `knowledge/thresholds/threshold_registry.json`): Zone (`room_1`/`room_2`/`corridor`), Device (rig equipment: mud pump, control panel, compressor, wellhead), Sensor, Manual, ThresholdSpec, FailureMode, Symptom, Action nodes.
- **Manual-derived thresholds:** limits come from device manuals, not hardcoded JSON. Offline: `knowledge/extraction/manual_threshold_extractor.py` (local LLM) extracts candidate ThresholdSpecs from `knowledge/documents/ONGC_Device_Manuals.txt` → human validates → `threshold_registry.json` → seeded into Neo4j. Runtime: `backend/services/threshold_resolver.py` resolves per-sensor limits with priority **device manual → zone environmental (HSE) → zone_definitions.json fallback** (also the safety net when Neo4j is down). Deterministic comparison in `backend/services/anomaly_evaluator.py` — the LLM never decides thresholds live. Inspect via `GET /api/thresholds`; re-resolve via `POST /api/thresholds/refresh` after re-seeding.
- **Trigger:** "RUN DIAGNOSTICS" button in the Sensor Console → `POST /api/diagnostics/run`. Backend threshold-checks **every zone** against current sensor data (using resolved thresholds) and publishes **one Kafka alert per flagged zone** (`rigvision_alerts`), each carrying a `threshold_context` explaining which device/manual limit fired. No flags → instant `all_clear`, no LLM call.
- **Pipeline:** `anomaly_listener` consumes each alert → KG Cypher query → ChromaDB vector search → LLM prompt → root-cause JSON. Each diagnostic is **self-describing** (carries its source `event_id`/`zone_id`/`severity`/telemetry/`threshold_context`) and lands in `rigvision:diagnostics` → AI Diagnostics modal.
- **Anti-hallucination:** the LLM schema has `anomaly_detected`; the prompt forces "No issue detected" when `triggered_sensors` is empty / data is within limits.
- **Models:** **Gemini** for embeddings only (`gemini-embedding-001`). **Answer generation runs locally** via LM Studio (OpenAI-compatible REST, called with `requests` — no `openai` SDK), configured by `LLM_BASE_URL`/`LLM_API_KEY`/`LLM_MODEL`. Zone→KG mapping: `zone_a→room_1`, `zone_b→room_2`, `corridor→corridor` (`_f1` variants reuse their base room).

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
| Vector Store | ChromaDB |
| LLM (embeddings) | Gemini (`gemini-embedding-001`) |
| LLM (generation) | Local via LM Studio (OpenAI-compatible REST), e.g. Qwen2.5-7B / Gemma |

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
[{"event_id": "anom_1716969600_zone_b", "zone_id": "zone_b", "severity": "CRITICAL", "anomaly_detected": true, "primary_diagnosis": "Motor Burnout", "confidence_score": 85, "reasoning": "...", "recommended_action": "...", "triggered_sensors": ["temperature"], "telemetry_snapshot": {"temperature": 68.0}, "timestamp": 1716969600000}]
```

---

## RUN ORDER (local dev)

Infra first: **Redis**, **Kafka**, **Neo4j**, **ChromaDB** (port 8100), and **LM Studio** (Developer → Start Server, model loaded, `n_parallel=1`).

1. **Backend API** — `python backend/main.py` (FastAPI on :8000; serves REST + WebSocket, hosts the Kafka producer for `/api/diagnostics/run` and the diagnostics consumer).
2. **CV pipeline** — `python cv/pipeline.py --mode demo` (fake people, real sensor feed; or `--mode live/video`). Writes `rigvision:persons` + `rigvision:zones`.
3. **Diagnostics listener** — `python knowledge/extraction/anomaly_listener.py` (Kafka → KG → LM Studio → `rigvision_diagnostics`).
4. **Frontend (dashboard)** — `cd frontend && npm run dev` → `:5173`.
5. **Frontend (sensor console)** — `cd frontend && npm run dev:sensors` → `:5174`.

Flow: Sensor Console sliders → SEND TO REDIS → RUN DIAGNOSTICS → per-zone Kafka alerts → LLM diagnoses → AI Diagnostics modal on the 3D dashboard.

---

## ZONE LAYOUT

```
   ┌────────────┐        ┌────────────┐
   │   ROOM A   │──│CORR│──│   ROOM B   │
   │  (zone_a)  │  │IDOR│  │  (zone_b)  │
   │   4×5m     │  │2×2m│  │   4×5m     │
   └────────────┘  └────┘  └────────────┘
   X: 0────4      4──6     6────10
```

Total: 10m × 5m × 3m. Origin at corner of Room A. Y = up.

---

## CONVENTIONS

- Python 3.11+, type hints encouraged
- Functional React components, hooks only
- Zustand for state, Three.js via @react-three/fiber
- Redis keys prefixed with `rigvision:`
- Coordinate system: X = length, Y = up, Z = width. Units = meters.
