# RigVision-3D — Runtime Architecture

Everything that runs, and how it runs: processes, threads, async tasks, locks, event loops, data flows, and timing.

---

## 1. Process Map

Five separate OS processes must be running simultaneously in production. None of them share memory; they communicate only through Redis, Kafka, and HTTP/WebSocket.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  PROCESS 1: Backend API         python backend/main.py                  │
│             Port :8000  (FastAPI + Uvicorn, asyncio event loop)         │
├─────────────────────────────────────────────────────────────────────────┤
│  PROCESS 2: CV Pipeline         python cv/pipeline.py --mode demo|live  │
│             No port; writes Redis directly                              │
├─────────────────────────────────────────────────────────────────────────┤
│  PROCESS 3: Anomaly Listener    python knowledge/extraction/            │
│                                          anomaly_listener.py            │
│             No port; reads Kafka, writes Redis + Kafka                  │
├─────────────────────────────────────────────────────────────────────────┤
│  PROCESS 4: Frontend Dashboard  npm run dev  →  :5173                   │
│             Vite dev server; browser connects via WS                    │
├─────────────────────────────────────────────────────────────────────────┤
│  PROCESS 5: Sensor Console      npm run dev:sensors  →  :5174           │
│             Separate Vite entry; no 3D, only sliders + diagnostics btn  │
└─────────────────────────────────────────────────────────────────────────┘

Supporting infrastructure (not application code):
  Redis 7              → shared memory bus
  Kafka                → reliable alert + diagnostics bus
  Neo4j 5.x            → knowledge graph queries
  ChromaDB  :8100      → vector search
  LM Studio :1234      → local LLM + embeddings (OpenAI-compatible REST)
```

---

## 2. Process 1 — Backend API (`backend/main.py`)

### Thread / Task Structure

```
uvicorn (main OS thread)
└── asyncio event loop
    ├── TASK: redis_to_websocket_bridge()         ← persistent loop, ~10 Hz
    ├── HANDLER (per connection): /ws/realtime     ← one coroutine per WS client
    └── HANDLER (per request): HTTP endpoints      ← one coroutine per HTTP req

OS thread  ← Kafka consumer daemon thread          ← started at app startup
            runs blocking consumer.poll(500 ms)
            on message → writes Redis synchronously
```

### Startup Sequence (serial)

```
lifespan() enters
  1. redis.ping()                         ← verify Redis is up
  2. publish_resolved_thresholds()        ← resolve device-manual thresholds,
                                            write rigvision:thresholds:resolved
  3. start_kafka_consumer()               ← spawn daemon thread on
                                            topic rigvision_diagnostics
  4. asyncio.create_task(
       redis_to_websocket_bridge())       ← schedule the broadcast loop
  5. yield                                ← app is live
  ... shutdown ...
  6. cancel bridge task
  7. close all WS connections
  8. disconnect Redis pool
```

### `redis_to_websocket_bridge()` — the real-time pump

Runs **forever** in the event loop, never blocks.

```
every 100 ms (asyncio.sleep(0.1)):
  ┌── asyncio.gather() [all four reads are concurrent, single await] ──┐
  │   r.get("rigvision:persons")                                        │
  │   r.get("rigvision:zones")                                          │
  │   r.get("rigvision:diagnostics")                                    │
  │   r.hgetall("rigvision:diag:progress")                              │
  └─────────────────────────────────────────────────────────────────────┘
  prune diag:progress entries older than 120 s
  build message dict
  manager.broadcast(json.dumps(msg))   ← sends to all active WS clients
                                          (each send is awaited in sequence)
```

### HTTP endpoint concurrency

All endpoints are `async def`. Blocking work (Kafka produce, heavy computation) is
offloaded with `asyncio.to_thread()` so the event loop is never blocked.

```
POST /api/diagnostics/run
  1. asyncio.Lock (_diagnostics_lock) → rate-limit check (30 s min interval)
  2. asyncio.to_thread(_evaluate_and_publish, sensors)
       inside the thread:
         for each zone:
           threshold_resolver.resolve()
           anomaly_evaluator.evaluate()
           if breach → kafka_producer.send("rigvision_alerts", ...)
  3. return {queued: n_alerts}
```

### Kafka consumer daemon thread (serial, separate OS thread)

```
while True:
  msgs = consumer.poll(timeout_ms=500)      ← blocks up to 500 ms
  for msg in msgs["rigvision_diagnostics"]:
    dedup by event_id (_seen_event_ids set)
    r.linsert / r.ltrim "rigvision:diagnostics"  ← prepend, keep 50
    consumer.commit()
```

This thread only writes Redis; it never touches the asyncio event loop directly.
The bridge task picks up the Redis write on its next 100 ms tick.

### Connection manager

`ConnectionManager` holds a plain Python `set` of active WebSocket objects.
`broadcast()` iterates the set and awaits each `ws.send_text()` in sequence
(not truly parallel — a slow client blocks the rest for that tick).
Disconnected clients are caught per-send and removed from the set.

---

## 3. Process 2 — CV Pipeline (`cv/pipeline.py`)

### Thread structure

```
main thread  (orchestrator + YOLO + Redis write)
├── CaptureThread × 4   (one per camera, daemon)
│     loop: cap.read() → put latest frame in shared slot
│           threading.Lock guards the slot
├── DisplayThread × 4   (one per camera, daemon)
│     loop: get latest frame → draw overlays → JPEG → Redis
│           reads LatestTracks (lock-free atomic swap)
└── PPEMonitor thread   (optional, every N frames)
      checks PPE compliance, writes rigvision:ppe:latest
```

Total threads in the CV process: **1 main + 4 capture + 4 display + 1 PPE = 10**.

### Main loop (serial per zone-group, ~10 Hz)

```
load calibration files (once at startup)
init YOLO model (once)
init Redis connection (once)
spawn CaptureThreads, DisplayThreads

loop forever:
  for zone_id in ["zone_a", "zone_b"]:         ← SERIAL — zones processed one at a time
    grab latest frame from each camera in zone  ← ThreadedCamera.read() under lock
    batch YOLO inference([frame_cam0, frame_cam1]) ← single GPU call, 2 images
    BoT-SORT tracker update per camera           ← sequential
    cross-camera identity match:
      1. ArUco marker IDs (if visible)
      2. epipolar geometry fallback
    DLT triangulation → (X, Y, Z) room coords
    merge into person list for zone

  collect all persons from both zones
  read rigvision:sensors:latest from Redis
  build_zone_states(persons, sensors)
  r.set("rigvision:persons", json)
  r.set("rigvision:zones",   json)
```

### CaptureThread (per camera)

```
while running:
  ret, frame = cap.read()            ← blocking VideoCapture.read()
  with lock:
    self.latest_frame = frame        ← atomic frame swap
    self.frame_seq += 1
```

Reading is lock-free from the main thread side (`ThreadedCamera.read()` acquires lock,
copies reference, releases). Frame buffer depth = 1 (always freshest frame; no queue backlog).

### DisplayThread (per camera)

```
while running:
  frame = camera.read()              ← lock, copy, release
  tracks = latest_tracks.get()       ← lock-free CAS via LatestTracks helper
  draw bboxes, labels, pose
  _, buf = cv2.imencode(".jpg", frame, [cv2.IMWRITE_JPEG_QUALITY, 60])
  r.setex(f"rigvision:camera:frame:{cam_id}", 2, buf.tobytes())
```

The display thread runs at camera FPS (capped ~25 Hz) independently of the main loop.
JPEG frames in Redis expire after 2 s; if the display thread dies, the MJPEG stream
in the browser shows a stale/blank frame automatically.

### Demo mode vs live mode

In `--mode demo` the pipeline generates synthetic persons with random walks
instead of running real camera capture. All other logic (triangulation, Redis writes,
zone-state building) is identical.

---

## 4. Process 3 — Anomaly Listener (`knowledge/extraction/anomaly_listener.py`)

### Thread structure

```
main thread  (Kafka consumer loop — blocking poll)
└── ThreadPoolExecutor(max_workers=3)   (LISTENER_WORKERS env var)
    ├── Worker A  →  Neo4j + ChromaDB + LLM  (for alert 1)
    ├── Worker B  →  Neo4j + ChromaDB + LLM  (for alert 2)
    └── Worker C  →  Neo4j + ChromaDB + LLM  (for alert 3)
```

Up to 3 diagnostic pipelines run **in parallel** (one per pool worker).
Workers share lazy singletons: Neo4j driver, ChromaDB client, LLM agent.

### Kafka consumer loop (main thread, serial)

```
for message in consumer:                ← blocks until message arrives
  pool.submit(_handle_message, msg)     ← non-blocking, returns Future
  # immediately loops back to await next Kafka message
  # pool workers run in background
```

If all 3 workers are busy, `pool.submit()` queues the task; the Kafka loop still
returns immediately (the queue is unbounded). `auto_offset_reset='latest'` means
a backlog of old alerts is never replayed after restart.

### Per-worker pipeline (serial within each worker)

```
_handle_message(alert):
  parse alert JSON
  emit progress: stage="generating_query" → Redis hash

  1. query_generator.build_cypher_params(alert)
     └── map zone_a→room_1, zone_b→room_2
         map triggered_sensor_type → symptom node
         low-breach → symptom "pressure_low" (not "pressure")

  emit progress: stage="getting_subgraph"

  2. graph_extractor.extract(params)
     └── Neo4j Cypher query (shared driver, thread-safe)
         topology_cache: static inter-device query is computed ONCE
                         per driver lifetime and cached
                         (threading.Lock guards the cache)

  emit progress: stage="subgraph_ready", subgraph=text

  3. diagnostic_agent.retrieve_manuals(zone_id, anomaly_text)
     └── embed query text → HTTP POST :1234/v1/embeddings (local LM Studio)
         ChromaDB.query(embedding, n_results=5)

  emit progress: stage="chunks_ready", chunks=text

  4. diagnostic_agent.generate_answer(subgraph, chunks, alert)
     └── build prompt (subgraph + manual excerpts + sensor context)
         HTTP POST :1234/v1/chat/completions (blocking, ~2–30 s on RTX 4070)
         parse JSON response schema

  emit progress: stage="writing_answer"

  5. kafka_producer.send("rigvision_diagnostics", report)
  emit progress: stage="done", report=full_json
```

### Progress emission (every stage)

Each stage update writes to the Redis hash `rigvision:diag:progress` under the
event's key. The hash expires after 600 s. The backend bridge reads it every 100 ms
and forwards it to all WebSocket clients as `diag_progress` in the `realtime_update`
message. The frontend `DiagnosticsLive` page renders the staged timeline live.

### Lazy singletons (shared across workers)

```python
_extractor: SubgraphExtractor | None = None   # Neo4j driver
_agent:     DiagnosticAgent   | None = None   # LLM agent + ChromaDB
_lock = threading.Lock()

def get_extractor():
    global _extractor
    with _lock:
        if _extractor is None:
            _extractor = SubgraphExtractor(...)
    return _extractor
```

All three workers call `get_extractor()` and `get_agent()` without re-creating
objects. Neo4j's official driver is thread-safe for concurrent sessions; ChromaDB's
Python client is safe for concurrent reads.

---

## 5. Frontend — Dashboard (`frontend/src/`)

The browser is single-threaded (one JS thread + one compositor thread in the GPU
process). Apparent concurrency comes from the event loop, WebSocket callbacks, and
the React reconciler.

### Boot sequence (serial, once)

```
main.jsx
  1. import authHandoff.js               ← runs immediately (before React mounts)
       reads window.opener.sessionStorage → copies API key into this tab
  2. ReactDOM.createRoot(...).render(<AppRouter />)

AppRouter.jsx
  3. <Routes>
       "/"           → <App />
       "/diagnostics" → <DiagnosticsLive />
     </Routes>

App.jsx (mounts)
  4. useEffect → useRigStore.connectToBackend()
       new WebSocket("ws://localhost:8000/ws/realtime")
       ws.onmessage = handler
       ws.onclose   = setTimeout(reconnect, 2000)
  5. requestAnimationFrame(rafLoop)      ← starts the render pump
```

### WebSocket message handler

```
ws.onmessage = (event) => {
  _latestRawData = JSON.parse(event.data)   ← store raw, do NOT update state yet
}
```

The handler is deliberately minimal — it only stashes the parsed payload.
State is applied in the RAF loop to prevent over-rendering.

### RAF loop (runs at browser refresh rate, capped by fpsLimit)

```
rafLoop(timestamp):
  elapsed = timestamp - lastFrameTime
  if elapsed >= (1000 / fpsLimit):          ← default 30 Hz cap
    if _latestRawData !== null:
      zustand.setState({
        persons:     _latestRawData.persons,
        zones:       _latestRawData.zones,
        diagnostics: _latestRawData.diagnostics,
        ppe:         _latestRawData.ppe,
        diagProgress:_latestRawData.diag_progress
      })
      _latestRawData = null
    lastFrameTime = timestamp
  requestAnimationFrame(rafLoop)            ← re-schedule
```

If the WebSocket sends 10 updates per second but fpsLimit is 30, each RAF tick
that has new data applies it; if the WS is slower (< 30 Hz), most RAF ticks are
no-ops (state unchanged, React bails out of reconciliation).

### Scene3D rendering (Three.js / @react-three/fiber)

```
@react-three/fiber manages its own RAF loop (separate from App's loop)
  reads Zustand state on each frame
  updates zone box colors (green/amber/red)
  animates person markers
  OrbitControls / PointerLockControls run on GPU compositor thread
```

Zone colors and person positions are derived directly from Zustand state; no
additional transforms happen inside Three.js.

### DiagnosticsLive page (`/diagnostics`)

```
Mounts in a separate browser tab (window.open from TopBar)
  1. authHandoff.js seeds API key from opener tab (runs before React)
  2. Same useRigStore connects its own WebSocket
  3. useEffect merges diagnostics + diagProgress into unified `events` list:
       - completed reports come from diagnostics array
       - in-flight stages come from diagProgress hash
       - merge key: event_id
  4. Left pane: sorted event list (newest first)
  5. Right pane:
       if event.stage !== "done":  → LivePipeline (staged timeline, auto-advances)
       else:                       → ReportDetail (telemetry grid, subgraph, reasoning)
```

---

## 6. Data Flow Summary

### Sensor alert → on-screen diagnosis (end-to-end)

```
User (Sensor Console :5174)
  │  drag slider, click "SEND TO REDIS"
  ▼
Redis: rigvision:sensors:latest
  │  (CV pipeline also reads this on every main-loop tick)
  ▼
User clicks "RUN DIAGNOSTICS"
  │  POST /api/diagnostics/run  (Backend :8000)
  ▼
Backend asyncio thread pool
  │  evaluate all zones (anomaly_evaluator.py)
  │  for each breached zone → kafka_producer.send("rigvision_alerts")
  ▼
Kafka: rigvision_alerts
  │
  ▼
Anomaly Listener (Process 3)
  │  consumer.poll() picks up alert
  │  pool.submit(_handle_message)  → Worker thread
  │    stage 1: query_generator.build_cypher_params
  │    stage 2: graph_extractor.extract  (Neo4j)
  │    stage 3: retrieve_manuals  (ChromaDB + LM Studio /v1/embeddings)
  │    stage 4: generate_answer   (LM Studio /v1/chat/completions)  ← slowest step
  │    stage 5: kafka_producer.send("rigvision_diagnostics")
  │
  │  (each stage writes rigvision:diag:progress hash)
  ▼
Kafka: rigvision_diagnostics
  │
  ▼
Backend Kafka consumer daemon thread
  │  dedup by event_id
  │  r.linsert("rigvision:diagnostics", report)
  ▼
Backend bridge task (next 100 ms tick)
  │  asyncio.gather reads rigvision:diag:progress + rigvision:diagnostics
  │  broadcast realtime_update to all WS clients
  ▼
Browser (Dashboard :5173 + Diagnostics tab)
  │  ws.onmessage stashes payload
  │  RAF loop applies to Zustand state
  │  DiagnosticsLive re-renders: LivePipeline → ReportDetail
```

### Live person tracking (continuous, ~10 Hz)

```
4 phone cameras (DroidCam RTSP)
  │
  ▼
CaptureThreads × 4  (one per camera, always running)
  │  cap.read() → latest_frame slot
  ▼
CV main loop (~10 Hz)
  │  grab frames from both cameras in zone
  │  YOLO batch inference (GPU)
  │  BoT-SORT track update
  │  cross-camera match (ArUco → epipolar)
  │  DLT triangulation
  ├─ zone_a done
  └─ zone_b done  (serial, not parallel)
  │
  │  read rigvision:sensors:latest
  │  build_zone_states(persons, sensors)
  │  r.set("rigvision:persons")
  │  r.set("rigvision:zones")
  ▼
DisplayThreads × 4  (parallel to main loop)
  │  draw overlays → JPEG → r.setex("rigvision:camera:frame:N", 2, ...)
  ▼
Backend bridge (next 100 ms tick)
  │  reads rigvision:persons, rigvision:zones
  │  broadcast realtime_update
  ▼
Browser
  │  RAF loop → Zustand → Scene3D re-renders person dots + zone colors
```

---

## 7. Concurrency & Locking Reference

| Object | Type | Owned by | Purpose |
|--------|------|----------|---------|
| `ThreadedCamera.lock` | `threading.Lock` | CV Process | Guards `latest_frame` + `frame_seq` slot |
| `LatestTracks._lock` | `threading.Lock` | CV Process | Guards track list shared between main loop and DisplayThread |
| `SubgraphExtractor._topology_lock` | `threading.Lock` | Listener Process | Ensures Neo4j topology is computed only once |
| `_singleton_lock` in listener | `threading.Lock` | Listener Process | Guards lazy-init of `_extractor` and `_agent` |
| `_diagnostics_lock` | `asyncio.Lock` | Backend Process | Rate-limits `/api/diagnostics/run` (30 s min) |
| `_redis_pool` | aioredis pool (20 conns) | Backend Process | Multiplexes async Redis reads across all coroutines |
| `ConnectionManager.active_connections` | Python `set` | Backend Process | WS client registry; mutated only inside coroutines (single event loop, no lock needed) |
| Kafka producer | `confluent_kafka.Producer` | Listener + Backend | Thread-safe by design; each process has its own instance |
| Kafka consumer | `confluent_kafka.Consumer` | Backend + Listener | One consumer per process; not shared across threads |

---

## 8. Update Rates & Timing

| Component | Rate | Notes |
|-----------|------|-------|
| CV pipeline main loop | ~10 Hz | Bound by YOLO GPU latency per batch |
| DisplayThread per camera | ~25 Hz | Bound by camera FPS |
| Redis bridge broadcast | 10 Hz | `asyncio.sleep(0.1)` |
| WebSocket messages received (browser) | 10 Hz | Matches bridge rate |
| RAF loop (browser) | Up to 30 Hz | Capped by `fpsLimit`; no-op if no new WS data |
| Three.js render | Up to 60 Hz | @react-three/fiber uses its own RAF |
| Kafka consumer poll timeout | 500 ms | Backend consumer thread; listener is event-driven |
| LLM generation latency | 2–30 s | Depends on prompt length and GPU contention |

---

## 9. What Is Parallel vs Serial

### Parallel (true concurrency)

- **CaptureThreads × 4** run simultaneously; each camera captures independently.
- **DisplayThreads × 4** run simultaneously; JPEG encode + Redis write is independent per camera.
- **Anomaly listener workers × 3** can run 3 full diagnostic pipelines simultaneously.
- **Backend asyncio tasks**: bridge loop, HTTP handlers, and WS handlers all run concurrently on the single event loop (cooperative multitasking — no two run at the same CPU instant, but none blocks the loop).
- **asyncio.gather** in the bridge makes 4 Redis reads issue concurrently before any result is awaited.

### Serial (strictly ordered)

- **CV zone-group processing**: zone_a is fully processed before zone_b starts. This was a design choice — per-zone-group fused independently, no shared frame state.
- **Within each anomaly listener worker**: query → Neo4j → ChromaDB → LLM generate → Kafka send — each step waits for the previous.
- **Backend startup**: Redis ping → threshold publish → Kafka consumer thread → bridge task. Must be ordered (bridge needs Redis; consumer needs Kafka).
- **Kafka message handling in the backend**: consumer.poll loop processes one message batch at a time, then commits.

### What shares resources and what is isolated

```
Neo4j driver      → shared by all 3 listener workers (driver is thread-safe)
ChromaDB client   → shared by all 3 listener workers
LLM HTTP calls    → shared GPU (LM Studio queues them; n_parallel=4 set in LM Studio)
Redis pool        → shared by all backend async coroutines (pool manages connections)
YOLO model        → used only by CV main thread (no sharing needed)
```

---

## 10. Failure Modes & Graceful Degradation

| Failure | Effect | Recovery |
|---------|--------|----------|
| Redis down | Backend returns 503; CV pipeline loop crashes | Redis reconnects on restart |
| Kafka down | Diagnostics queue blocks; alert publish fails silently | Manual diagnostics unavailable; sensor display unaffected |
| Neo4j down | Listener worker raises; logs error; emits `stage=error` | Threshold resolver falls back to `zone_definitions.json` |
| LM Studio down | `generate_answer` HTTP call fails; worker catches exception | Progress shows `stage=error`; no diagnosis report emitted |
| ChromaDB down | `retrieve_manuals` returns empty; LLM gets no manual context | Diagnosis proceeds with KG-only context (degraded quality) |
| CV pipeline crash | `rigvision:persons` and `rigvision:zones` go stale; dashboard shows last known state | Restart pipeline; bridge auto-picks up new data |
| WebSocket disconnect | Browser reconnects after 2 s; 100 ms bridge cycle means at most ~100 ms of data loss | Auto-reconnect in `useRigStore` |
| MJPEG frame stale | `rigvision:camera:frame:N` TTL 2 s; browser shows blank frame | DisplayThread restart restores stream |
