# Backend API Walkthrough

Main file: `backend/main.py`

The backend has four jobs:

1. Read current state from Redis.
2. Broadcast state to the frontend over WebSocket.
3. Serve latest camera frames as MJPEG streams.
4. Consume Kafka diagnostic events and store them in Redis.

## Imports And Logging

`asyncio`, `json`, `time`, `base64`, and `os` are standard-library tools for async loops, serialization, timestamps, image decoding, and configuration.

`redis.asyncio` is used for async Redis calls in request handlers and the WebSocket bridge.

`FastAPI`, `WebSocket`, `HTTPException`, and `StreamingResponse` are the web framework pieces.

`pydantic.BaseModel` is used for validating the VLM gating request body.

Why async:

- WebSockets and MJPEG streams are long-lived.
- Async lets the server handle many clients without one thread per client.

## Redis Connection Setup

The backend reads `REDIS_HOST`, `REDIS_PORT`, and `REDIS_PASSWORD` from environment variables.

The `_get_pool()` helper lazily creates one Redis connection pool.

The `get_redis()` helper returns a Redis client using that pool.

Why a connection pool:

- Request handlers should not create a new TCP connection for every Redis call.
- Pooling keeps latency lower and resource use predictable.

## `ConnectionManager`

This class owns all active WebSocket clients.

`connect(websocket)`:

- Accepts the WebSocket handshake.
- Adds the client to `active_connections`.
- Logs the current client count.

`disconnect(websocket)`:

- Removes the client safely.
- Logs the current client count.

`broadcast(message)`:

- Sends one message to every connected client.
- Uses `asyncio.gather` so clients are written concurrently.
- Removes dead clients if sending fails.

`close_all()`:

- Closes clients during server shutdown.

Why a manager class:

- WebSocket state is shared by the bridge loop and request handlers.
- A class keeps this state localized instead of scattering globals.

## Redis-To-WebSocket Bridge

Function: `redis_to_websocket_bridge()`

Workflow:

1. Create a Redis client.
2. Loop forever.
3. If any frontend clients are connected, fetch:
   - `rigvision:persons`
   - `rigvision:zones`
   - `rigvision:violations:latest`
   - `rigvision:diagnostics`
4. Hash the raw Redis strings.
5. If the hash changed, parse JSON and broadcast one combined `realtime_update`.
6. Sleep for 0.1 seconds.

Why compare raw hashes:

- Avoids broadcasting unchanged state.
- Avoids deep object comparisons.
- Redis returns strings, so hashing the tuple is cheap.

Why combine all keys:

- The frontend receives one coherent update payload.
- Zustand can update related state together.

## Kafka Consumer Thread

Function: `start_kafka_consumer()`

This starts a daemon thread because the main app is async, while `kafka-python` uses a blocking consumer style.

It subscribes to:

- `rigvision_alerts`
- `rigvision_diagnostics`

When it sees an alert:

- Stores it in `rigvision:latest_alert`.
- Keeps it in memory as `latest_alert`.

When it sees a diagnostic:

- Parses the diagnostic.
- Merges alert metadata into it.
- Inserts it at the front of `rigvision:diagnostics`.
- Keeps only the latest 50 entries.

Why merge alert metadata:

- The LLM diagnostic may only contain reasoning.
- The frontend needs zone, severity, triggered sensors, and telemetry snapshot too.

Why thread instead of async Kafka:

- It avoids adding another async Kafka dependency.
- It isolates blocking Kafka polling from FastAPI's event loop.

## Lifespan Hook

The `lifespan` async context manager:

1. Pings Redis on startup.
2. Starts the Kafka consumer thread.
3. Starts the Redis-to-WebSocket bridge task.
4. On shutdown, cancels the bridge, closes WebSockets, and disconnects Redis pool.

Why lifespan:

- Startup/shutdown logic belongs with the app lifecycle.
- It prevents orphaned WebSocket tasks on exit.

## HTTP Endpoints

`GET /api/health`

- Pings Redis.
- Reports backend status, Redis status, connected WebSocket clients, and timestamp.

`GET /api/persons`

- Reads `rigvision:persons`.
- Returns parsed array or empty list.

`GET /api/zones`

- Reads `rigvision:zones`.
- Returns parsed object or empty object.

`GET /api/violations`

- Reads `rigvision:violations:latest`.
- Returns parsed array or empty list.

`GET /api/diagnostics`

- Reads `rigvision:diagnostics`.
- Returns parsed array or empty list.

Why direct HTTP endpoints if WebSocket exists:

- Useful for debugging.
- Useful for initial load if WebSocket is unavailable.
- Easy to inspect with a browser or curl.

## MJPEG Camera Endpoint

Endpoint: `GET /api/video/mjpeg/{camera_id}`

Workflow:

1. Check active stream limit.
2. Check if `rigvision:camera:frame:<camera_id>` exists.
3. Start an async generator.
4. Repeatedly read base64 JPEG from Redis.
5. Decode to bytes.
6. Yield multipart MJPEG frame chunks.
7. Stop if frames are missing for too long.

Why MJPEG:

- Simple browser support through `<img src=...>`.
- No WebRTC setup.
- Good enough for demo and monitoring overlays.

Why Redis frame TTL:

- If CV pipeline stops, old camera frames should disappear.
- The backend can report camera offline instead of showing stale evidence.

## Control Endpoints

`POST /api/control/vlm_gating`

- Accepts JSON body `{ "enabled": true/false }`.
- Writes setting to Redis.

`POST /api/control/clear_cache`

- Publishes `clear_cache` on Redis channel `rigvision:commands`.
- The CV pipeline listens and resets trackers/recognition caches.

Why Redis pub/sub for commands:

- The backend does not call the CV process directly.
- Any process can listen to the command channel.

## WebSocket Endpoint

Endpoint: `/ws/realtime`

Workflow:

1. Accept client.
2. Keep receiving messages.
3. If frontend sends `{ "type": "ping" }`, reply with `{ "type": "pong" }`.
4. On disconnect/error, remove client.

Why receive messages if backend mostly pushes:

- Keeps the connection active.
- Allows simple ping/pong health checks.
- Leaves room for future client commands.

