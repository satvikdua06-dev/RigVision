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
- Cross-camera matching uses ReID appearance similarity + epipolar geometry
- DLT triangulation converts 2D pixel pairs → 3D room coordinates
- Output: JSON array of tracked persons written to Redis at ~10Hz

### Pillar 3: Sensor Fusion
- Sensor types: temperature, vibration (g RMS), noise (dB), gas (H₂S ppm), pressure
- MQTT protocol via EMQX broker
- Anomaly detection: absolute thresholds, Z-score > 3σ from rolling baseline
- PostgreSQL + TimescaleDB for time-series storage
- Redis for current zone states

### Pillar 4: Compliance Engine
- Rule-based system evaluating safety rules every 2 seconds
- Rules defined in YAML (PPE requirements, occupancy limits, environmental thresholds)
- Violations logged to PostgreSQL with evidence frames

### Pillar 5: Knowledge Graph + LLM Reasoning
- Neo4j graph: Zone, Equipment, Sensor, FailureMode, Symptom, Procedure nodes
- RAG pipeline: KG Cypher query → ChromaDB vector search → LLM prompt → root-cause response

---

## TECH STACK

| Layer | Tech |
|-------|------|
| 3D Rendering | Three.js, @react-three/fiber, @react-three/drei |
| UI Framework | React 18, Zustand, Vite |
| Backend API | FastAPI, Uvicorn |
| CV Detection | YOLOv8l (Ultralytics), RTMPose-L |
| CV Tracking | BoT-SORT (via boxmot), IoU fallback |
| Video Ingest | OpenCV, FFmpeg |
| Sensor Transport | EMQX (MQTT) |
| Database | PostgreSQL 16 + TimescaleDB |
| Cache | Redis 7 |
| Knowledge Graph | Neo4j 5.x |
| Vector Store | ChromaDB |
| LLM | Gemini 1.5 Flash / Codex Sonnet |

---

## REDIS DATA CONTRACTS

### `rigvision:persons` (CV pipeline → Redis at ~10Hz)
```json
[{"id": 1, "x": 3.2, "y": 0.05, "z": 2.5, "zone": "zone_a", "posture": "standing", "ppe": {"hardhat": true, "vest": false, "goggles": false}, "confidence": 0.91, "cameras_visible": 1}]
```

### `rigvision:zones` (sensor engine → Redis at ~1Hz)
```json
{"zone_a": {"status": "normal", "temperature": 28.3, "vibration": 1.2, "noise": 72, "gas_h2s": 0.5, "person_count": 2, "ppe_violations": [], "updated_at": 1716969600}}
```

### `rigvision:violations:latest` (compliance engine → Redis)
```json
[{"id": "v-001", "rule_id": "PPE-001", "zone": "zone_b", "severity": "HIGH", "message": "Person #3 missing hard hat", "person_ids": [3], "timestamp": 1716969600}]
```

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
