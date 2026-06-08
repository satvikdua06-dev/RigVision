# System Workflow

This document explains the full RigVision-3D workflow before diving into individual files.

## Big Picture

The project is a real-time monitoring dashboard. It combines:

- Computer vision: where people are, what PPE they are wearing, and which cameras see them.
- Sensor telemetry: temperature, vibration, noise, H2S gas, and pressure.
- Compliance rules: PPE and environmental violations.
- Knowledge reasoning: graph and RAG-backed diagnosis for abnormal events.
- 3D UI: a browser-based digital twin showing rooms, persons, sensors, cameras, and incident reports.

The important design choice is decoupling. Each subsystem can run independently and communicate through Redis or Kafka.

## Runtime Data Flow

```text
Phone cameras / videos
        |
        v
cv/pipeline.py
        |
        | writes rigvision:persons
        | writes rigvision:zones
        | writes rigvision:camera:frame:<camera_id>
        v
Redis
        ^
        | writes sensor-updated rigvision:zones
sensors/ingest/kafka_bridge.py
        ^
        |
sensors/simulator/simulate.py -> Kafka topic rigvision.sensors

Redis rigvision:persons + rigvision:zones
        |
        v
sensors/compliance/engine.py
        |
        | writes rigvision:violations:latest
        v
Redis

Kafka rigvision_alerts
        |
        v
knowledge/extraction/anomaly_listener.py
        |
        | Neo4j query + ChromaDB retrieval + Gemini response
        | publishes rigvision_diagnostics
        v
backend/main.py Kafka thread
        |
        | writes rigvision:diagnostics
        v
Redis

backend/main.py
        |
        | WebSocket /ws/realtime
        | MJPEG /api/video/mjpeg/<camera_id>
        v
frontend React app
```

## Why Redis For Live State?

Redis is used as the real-time state hub because:

- The CV pipeline can overwrite current positions at about 10 Hz without managing database rows.
- The frontend only needs the latest state, not the entire history.
- The backend can cheaply read several keys and broadcast one combined payload.
- Developers can debug with `redis-cli get rigvision:persons`.

PostgreSQL and TimescaleDB are more appropriate for historical storage. Redis is better for current dashboard state.

## Why Kafka For Events?

Kafka is used for sensor and diagnostic events because:

- Event streams can be replayed or consumed by more than one service.
- Sensor simulation and anomaly listeners can run independently.
- A future production system can add persistence, dashboards, or alerting consumers without rewriting the producer.

Redis answers "what is true right now?" Kafka answers "what happened?"

## Why A Browser Frontend?

The frontend is browser-based because:

- Three.js can render the 3D model without a native app.
- React state updates pair naturally with WebSocket data.
- Operators and reviewers can open the same dashboard from any machine on the network.

## End-To-End Demo Workflow

1. Start infrastructure with Docker Compose.
2. Start the backend.
3. Start `cv/pipeline.py` in demo, video, or live mode.
4. Optionally start sensor simulator and Kafka bridge.
5. Optionally start compliance engine.
6. Open the frontend.
7. Backend broadcasts real-time updates every time Redis state changes.

## Core Redis Keys

`rigvision:persons`

- Written by CV pipeline.
- Array of tracked people.
- Each person has position, zone, posture, PPE, confidence, camera visibility, and optional recognition method.

`rigvision:zones`

- Written by CV pipeline in demo/live/video modes and also updated by the sensor bridge.
- Object keyed by zone ID.
- Each zone has status, warning reason, telemetry, person count, PPE violation summaries, and update time.

`rigvision:violations:latest`

- Written by compliance engine.
- Array of current safety violations.

`rigvision:diagnostics`

- Written by backend Kafka consumer after knowledge layer publishes diagnostics.
- Array of recent root-cause diagnostic reports.

`rigvision:camera:frame:<camera_id>`

- Written by CV pipeline as base64 JPEG.
- Read by backend MJPEG endpoint.

## Common Failure Modes

- Zone ID mismatch: one service emits `room_1` while another expects `zone_a`.
- Timestamp unit mismatch: some fields use seconds, others milliseconds.
- Missing Redis: backend and CV pipeline fail early or reconnect poorly.
- Frontend stale assumptions: frontend mirror of zones drifts from `cad/zone_definitions.json`.
- Model mismatch: COCO YOLO models detect people only, while PPE models detect hardhat/vest/goggles too.

