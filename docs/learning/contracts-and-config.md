# Contracts And Configuration

This document explains the files that define shared project structure: zones, Redis payloads, Docker services, database schema, and frontend/build configuration.

## `cad/zone_definitions.json`

Purpose: canonical physical layout of the digital twin.

Important sections:

- Top-level metadata defines schema, title, description, and version.
- `room.total_dimensions` describes the full space.
- `zones` defines each named spatial region.
- Each zone has `bounds.min` and `bounds.max` in meters.
- Equipment entries define visual/semantic objects inside a zone.
- Sensors define telemetry source positions and warning/critical thresholds.

Why this file exists:

- The CV pipeline needs bounds to assign people to zones.
- The frontend needs geometry positions for overlays, sensors, cameras, and equipment.
- The compliance engine needs zone IDs and policy context.

Why JSON:

- Easy to load from Python and JavaScript.
- Easy to validate later against a JSON schema.
- Better than hard-coding coordinates in each subsystem.

Watch out:

- `frontend/src/utils/zonePositions.js` mirrors this data manually. That is convenient for the current UI, but it can drift from this canonical JSON.
- Floor-1 zones are present here, so downstream contracts must include `zone_a_f1`, `corridor_f1`, and `zone_b_f1`.

## `contracts/redis-schemas.json`

Purpose: documents intended Redis payload shapes.

Important sections:

- `definitions.ppe_status`: shape of hardhat/vest/goggles status.
- `definitions.tracked_person`: shape of each person in `rigvision:persons`.
- `definitions.zone_state`: shape of each zone in `rigvision:zones`.
- `definitions.violation`: shape of each compliance violation.
- `redis_keys`: maps Redis keys to schemas.

Why this file exists:

- Multiple services communicate through Redis without direct function calls.
- A schema lets the team agree on field names and types.
- It can become the basis for automated validation tests.

Current caveat:

- The schema is behind the runtime code. The code emits floor-1 zones, `floor`, `recognition_method`, and nullable PPE states. The schema still lists only the original three zones and boolean-only PPE.

Why this matters:

- Frontend, backend, and compliance can all appear correct individually while disagreeing on payloads.
- Contract drift is the fastest way to get demo-day bugs.

## `docker-compose.yml`

Purpose: starts infrastructure services.

Services:

- `redis`: current real-time state.
- `postgres`: relational/time-series store.
- `neo4j`: knowledge graph.
- `chromadb`: vector store for manuals/RAG.
- `zookeeper` and `kafka`: event streaming.

Why Docker Compose:

- Everyone on the team gets the same ports and credentials.
- Databases are isolated from local machine setup.
- Containers can be restarted independently during development.

Watch out:

- Kafka has internal and host listeners. Code running on the host uses `localhost:9092`; code inside Docker would use `kafka:29092`.
- Credentials are dev credentials. Do not use these as production secrets.

## `infra/postgres/init.sql`

Purpose: initializes relational tables and TimescaleDB structures.

Typical responsibilities in this project:

- Create extension support.
- Create sensor reading tables.
- Create violation/audit tables.
- Add indexes for zone/time queries.

Why PostgreSQL/TimescaleDB:

- Redis only stores current state.
- Historical sensor trends, compliance logs, and incident audits need durable storage.
- TimescaleDB is built for time-series data.

## `frontend/package.json`

Purpose: declares frontend dependencies and scripts.

Key scripts:

- `dev`: starts Vite dev server.
- `build`: creates production bundle.
- `lint`: runs ESLint.
- `preview`: previews production build.

Key dependencies:

- `react` and `react-dom`: UI.
- `zustand`: global state store.
- `three`, `@react-three/fiber`, `@react-three/drei`: 3D rendering.
- `recharts`: charting.
- `lucide-react`: icons.

Why Vite:

- Fast local dev server.
- Simple React setup.
- Good static asset handling for GLTF and textures.

## `frontend/vite.config.js`

Purpose: configures Vite.

Typical responsibilities:

- Load React plugin.
- Load Tailwind plugin.
- Set dev server options if needed.

Why a build config:

- Keeps frontend tooling explicit.
- Lets the team add aliases, proxy rules, or plugin settings later.

## `frontend/eslint.config.js`

Purpose: defines lint rules.

Why linting matters here:

- React hooks are easy to misuse.
- The frontend is highly stateful and visual.
- Lint catches unused imports, impure render logic, and questionable effects before demo time.

## `Makefile`

Purpose: developer shortcuts.

Important targets:

- `up`: start Docker infrastructure.
- `backend`: run FastAPI app.
- `frontend`: run Vite app.
- `cv-demo`: run simulated CV mode.
- `cv-live`: run live camera mode.
- `sensor-sim`: publish fake telemetry.
- `kafka-bridge`: move Kafka sensor events into Redis.
- `compliance`: evaluate safety rules.
- `clean`: remove generated/runtime artifacts.

Why a Makefile:

- Keeps command knowledge out of memory.
- Makes onboarding easier.
- Gives the team shared language for running services.

