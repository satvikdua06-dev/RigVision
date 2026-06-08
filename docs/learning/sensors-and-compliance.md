# Sensors And Compliance Walkthrough

This document explains sensor simulation, Kafka ingestion, and rule evaluation.

## `sensors/simulator/simulate.py`

Purpose: publish fake sensor readings to Kafka for development and demos.

Imports:

- `json`, `logging`, `math`, `os`, `signal`, `sys`, `threading`, `time`.
- `numpy` for random noise.
- `KafkaProducer` imported lazily inside runtime.

Why lazy Kafka import:

- The file can show a helpful error if `kafka-python` is missing.
- Importing the module does not immediately require Kafka dependencies.

Global state:

- `shutdown_event` lets Ctrl+C stop the loop.
- `ZONES` lists floor 0 and floor 1 zone IDs.

`generate_reading(zone_id, t)`

Workflow:

1. Compute a deterministic zone offset.
2. Generate temperature, vibration, noise, H2S, and pressure using sine waves.
3. Add random noise.
4. Inject occasional anomalies for demo interest.
5. Return one reading payload.

Why sine waves plus noise:

- Real sensor data changes smoothly but not perfectly.
- Sine waves are simple fake baselines.
- Noise makes the dashboard feel alive.

`run_simulator(...)`

Workflow:

1. Create Kafka producer.
2. Loop until shutdown.
3. Generate one reading for each zone.
4. Publish to topic `rigvision.sensors`.
5. Sleep according to requested rate.

Why publish one message per zone:

- The bridge can update zones independently.
- This matches how real distributed sensors would report.

## `sensors/ingest/kafka_bridge.py`

Purpose: consume sensor events from Kafka and update Redis zone state.

Constants:

- `SF`: sensor fields.
- `TH`: warning/critical thresholds.

`determine_sensor_status(readings)`

Workflow:

1. Start with `normal`.
2. For each reading, compare against critical then warning threshold.
3. Return highest status and reason.

Why central helper:

- Keeps threshold logic out of the Kafka loop.

`run_bridge(...)`

Workflow:

1. Connect to Redis.
2. Create Kafka consumer for sensor topic.
3. Poll Kafka.
4. For each message:
   - parse zone ID,
   - extract numeric readings,
   - load current `rigvision:zones`,
   - update that zone's readings,
   - compute status,
   - preserve person count/PPE violation fields,
   - write zones object back to Redis.

Why read-modify-write:

- The sensor message updates telemetry, but the zone state also contains person count and PPE data.

Tradeoff:

- CV pipeline and sensor bridge can both write `rigvision:zones`, so last writer wins.
- A more robust design would split keys, for example `rigvision:zone:sensors` and `rigvision:zone:occupancy`.

## `sensors/compliance/rules/ppe_rules.yaml`

Purpose: declarative safety rules.

Rule types:

- `ppe`: require PPE items in specified zones.
- `occupancy`: max people in a zone.
- `environment`: telemetry threshold rules.

Why YAML:

- Safety rules can be adjusted without editing Python.
- Easier for non-programmers to read than code.

## `sensors/compliance/engine.py`

Purpose: periodically evaluate safety rules and publish current violations.

`load_rules(rules_dir)`

Workflow:

1. Import PyYAML.
2. Scan rules directory.
3. Load `.yaml` and `.yml` files.
4. Collect all entries under `rules`.

Why multiple rule files:

- PPE, occupancy, and environmental rules can be separated later.

`evaluate_ppe_rule(rule, persons, zones)`

Workflow:

1. Read required PPE list and allowed zones.
2. For each person in matching zones:
   - inspect `person.ppe`,
   - if required item is explicitly `False`, create violation.

Why explicit `False`:

- `None` means unknown/unmonitored.
- Treating unknown as violation may be too aggressive for demos.

`evaluate_occupancy_rule(rule, persons, zones)`

Workflow:

1. Count person IDs by zone.
2. If count exceeds max occupancy, create violation.

Why count from persons:

- CV pipeline owns current occupancy.

`evaluate_environment_rule(rule, persons, zones)`

Workflow:

1. Read sensor type and threshold.
2. For each matching zone, read telemetry.
3. If value exceeds threshold, create violation with people present in that zone.

Why include people:

- Incident response needs to know who may be exposed.

`RULE_EVALUATORS`

- Maps rule type strings to evaluator functions.

Why function dispatch:

- Adding a new rule type means writing one function and adding it to the map.

`run_engine(...)`

Workflow:

1. Load YAML rules.
2. Connect to Redis.
3. Every interval:
   - read `rigvision:persons`,
   - read `rigvision:zones`,
   - evaluate all rules,
   - write `rigvision:violations:latest`,
   - log summary every 10 cycles.

Why every 2 seconds:

- Compliance does not need 10 Hz updates.
- A slower loop reduces noisy repeated violations.

## Scripts

`scripts/record_test_video.py`

- Utility for recording input video.
- Useful for reproducible testing without live cameras.

`scripts/test_yolo_video.py`

- Utility for testing YOLO inference on a video.
- Useful before integrating into the full pipeline.

