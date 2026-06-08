# Source File Index

This index maps project files to the learning resources that explain them.

## Backend

`backend/main.py`

- Explained in `backend-api.md`.
- Key topics: FastAPI app lifecycle, Redis pooling, WebSocket broadcasting, Kafka diagnostic consumer, MJPEG streaming, control endpoints.

`backend/requirements.txt`

- Dependency list for the API service.
- Read with `backend-api.md` to understand why FastAPI, Redis, Kafka, and Uvicorn are required.

## Computer Vision

`cv/pipeline.py`

- Explained in `cv-pipeline.md`.
- Key topics: demo/live/video modes, Redis writes, frame upload, identity propagation, cross-camera fusion, zone state generation.

`cv/detection/detector.py`

- Explained in `cv-submodules.md`.
- Key topics: YOLO loading, PPE model detection, PPE association, posture estimation, face/QR recognition.

`cv/tracking/tracker.py`

- Explained in `cv-submodules.md`.
- Key topics: BoT-SORT wrapper, IoU metadata transfer, tracked person model.

`cv/tracking/cross_camera.py`

- Explained in `cv-submodules.md`.
- Key topics: global IDs, appearance distance, epipolar distance, cross-camera matching.

`cv/tracking/triangulation.py`

- Explained in `cv-submodules.md`.
- Key topics: camera calibration objects, DLT triangulation, ground-plane fallback, zone assignment.

`cv/calibration/calibrate_intrinsic.py`

- Explained in `cv-submodules.md`.
- Key topics: checkerboard corner detection, camera matrix, distortion coefficients.

`cv/calibration/calibrate_extrinsic.py`

- Explained in `cv-submodules.md`.
- Key topics: stereo calibration, rotation, translation, fundamental matrix.

`cv/tracking/botsort/*.py`

- Explained in `cv-submodules.md`.
- Key topics: base track state, Kalman filtering, matching, BoT-SORT update loop.

`cv/requirements.txt`

- Dependency list for the CV runtime.
- Read with `cv-submodules.md` to understand why OpenCV, Ultralytics, NumPy, SciPy, and tracking dependencies are needed.

`cv/__init__.py`, `cv/detection/__init__.py`, `cv/tracking/__init__.py`, `cv/calibration/__init__.py`

- Package marker files.
- They are intentionally empty or minimal.

## Frontend

`frontend/src/main.jsx`

- Explained in `frontend.md`.
- Key topics: React root bootstrap.

`frontend/src/App.jsx`

- Explained in `frontend.md`.
- Key topics: top-level layout and backend connection.

`frontend/src/stores/useRigStore.js`

- Explained in `frontend.md`.
- Key topics: Zustand state, WebSocket reconnect, render decoupling, control actions.

`frontend/src/components/Scene3D.jsx`

- Explained in `frontend.md`.
- Key topics: Three.js scene, room GLTF loading, corridor geometry, settings panel, render throttling.

`frontend/src/components/ZonePlane.jsx`

- Explained in `frontend.md`.
- Key topics: zone overlays, status colors, selection, telemetry popup.

`frontend/src/components/PersonAvatar.jsx`

- Explained in `frontend.md`.
- Key topics: tracked person rendering, movement smoothing, PPE alert visuals.

`frontend/src/components/CameraFeeds.jsx`

- Explained in `frontend.md`.
- Key topics: selected-person camera evidence, MJPEG image streams, offline retry state.

`frontend/src/components/Sidebar.jsx`

- Explained in `frontend.md`.
- Key topics: zone/person/violation tabs, telemetry bars, searches, toggles.

`frontend/src/components/TopBar.jsx`

- Explained in `frontend.md`.
- Key topics: global status counts, diagnostics entry point.

`frontend/src/components/DiagnosticsModal.jsx`

- Explained in `frontend.md`.
- Key topics: incident list, LLM diagnostic rendering, telemetry snapshot, recommended actions.

`frontend/src/components/CameraIndicator.jsx`

- Explained in `frontend.md`.
- Key topics: camera markers in 3D.

`frontend/src/components/SensorIndicator.jsx`

- Explained in `frontend.md`.
- Key topics: sensor markers and hover labels.

`frontend/src/utils/zonePositions.js`

- Explained in `frontend.md` and `contracts-and-config.md`.
- Key topics: frontend geometry mirror of CAD layout.

`frontend/src/index.css`, `frontend/src/App.css`

- Styling files.
- Read with `frontend.md` after understanding the component structure.

`frontend/package.json`, `frontend/vite.config.js`, `frontend/eslint.config.js`

- Explained in `contracts-and-config.md`.
- Key topics: dependencies, build tooling, lint rules.

## Sensors And Compliance

`sensors/simulator/simulate.py`

- Explained in `sensors-and-compliance.md`.
- Key topics: fake telemetry generation, Kafka publishing, smooth noisy sensor baselines.

`sensors/ingest/kafka_bridge.py`

- Explained in `sensors-and-compliance.md`.
- Key topics: Kafka consumer, Redis zone updates, threshold status calculation.

`sensors/compliance/engine.py`

- Explained in `sensors-and-compliance.md`.
- Key topics: YAML rule loading, PPE rules, occupancy rules, environment rules, violation publishing.

`sensors/compliance/rules/ppe_rules.yaml`

- Explained in `sensors-and-compliance.md`.
- Key topics: declarative rule format.

## Knowledge And Diagnostics

`knowledge/graph/seed_graph.py`

- Explained in `knowledge-and-diagnostics.md`.
- Key topics: Neo4j topology seed, zones, devices, failures, symptoms, actions.

`knowledge/extraction/query_generator.py`

- Explained in `knowledge-and-diagnostics.md`.
- Key topics: anomaly payload parsing, parameterized Cypher template.

`knowledge/extraction/graph_extractor.py`

- Explained in `knowledge-and-diagnostics.md`.
- Key topics: Cypher execution, LLM context formatting.

`knowledge/extraction/anomaly_listener.py`

- Explained in `knowledge-and-diagnostics.md`.
- Key topics: Kafka alert listener, graph extraction, LLM report generation, diagnostic publishing.

`knowledge/agent_layer/rag_ingestion.py`

- Explained in `knowledge-and-diagnostics.md`.
- Key topics: manual ingestion, embeddings, ChromaDB collection.

`knowledge/agent_layer/diagnostic_agent.py`

- Explained in `knowledge-and-diagnostics.md`.
- Key topics: Gemini calls, Chroma retrieval, strict JSON response prompt.

`knowledge/trigger.py`

- Explained in `knowledge-and-diagnostics.md`.
- Key topics: manual Kafka diagnostic trigger.

`knowledge/documents/ONGC_Device_Manuals.txt`

- RAG source text for diagnostic recommendations.
- Read after `knowledge-and-diagnostics.md`.

`knowledge/requirements.txt`

- Dependency list for the knowledge layer.

## Contracts, Data, And Infrastructure

`cad/zone_definitions.json`

- Explained in `contracts-and-config.md`.
- Key topics: room geometry, zones, equipment, sensors, thresholds.

`contracts/redis-schemas.json`

- Explained in `contracts-and-config.md`.
- Key topics: shared Redis payload definitions and current drift.

`infra/postgres/init.sql`

- Explained in `contracts-and-config.md`.
- Key topics: durable relational/time-series storage setup.

`docker-compose.yml`

- Explained in `contracts-and-config.md`.
- Key topics: Redis, PostgreSQL/TimescaleDB, Neo4j, ChromaDB, Kafka, Zookeeper.

`Makefile`

- Explained in `contracts-and-config.md`.
- Key topics: shared developer commands.

## Utility Scripts

`scripts/record_test_video.py`

- Explained briefly in `sensors-and-compliance.md`.
- Purpose: record test video for repeatable CV experiments.

`scripts/test_yolo_video.py`

- Explained briefly in `sensors-and-compliance.md`.
- Purpose: test YOLO on saved footage before running full pipeline.

