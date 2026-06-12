# Anomaly Detector — Detailed Context & Handoff

**Status:** Implemented (2026-06-12). Threshold registry: `knowledge/thresholds/threshold_registry.json`; extractor: `knowledge/extraction/manual_threshold_extractor.py`; KG seeding: `knowledge/graph/seed_graph.py`; runtime: `backend/services/threshold_resolver.py` + `backend/services/anomaly_evaluator.py`, wired into `POST /api/diagnostics/run` (alerts now carry `threshold_context`). Inspect resolved limits via `GET /api/thresholds`.
**Owner:** RigVision-3D team
**Purpose:** Self-contained brief detailing the future direction for transition from temporary placeholder thresholds to manual-derived, device-aware anomaly detection resolved via the Neo4j knowledge graph.

---

## 1. Project Context & The Current Shortcut

RigVision-3D is a real-time 3D digital-twin for monitoring ONGC drilling rigs. The anomaly detector is currently implemented as an **on-demand sensor-threshold diagnostic pipeline**. In the present implementation, the frontend allows the operator to manually set live sensor values, the backend stores those values in Redis, and when the operator clicks **Run Diagnostics**, the backend checks those values against hardcoded warning and critical thresholds defined in `cad/zone_definitions.json`.

However, these hardcoded thresholds are only a **temporary development shortcut**.

The intended final design is more advanced and more industrially realistic: the system should not depend on generic fixed thresholds written manually in a JSON file. Instead, it should derive device-specific operating limits from the **actual equipment manuals**, connect those limits to physical devices through the **knowledge graph**, map devices to rooms/zones, and then use the correct threshold for each sensor depending on which device, room, sensor type, and operating condition are involved.

So the long-term goal is:

```text
Manual-derived thresholds
+ device-to-room knowledge graph mapping
+ live sensor values
= context-aware anomaly detection
```

The LLM should not be the raw anomaly detector. The LLM’s role should remain diagnostic and interpretive. The actual anomaly decision should still be made through deterministic logic, but the thresholds used by that logic should come from verified engineering documentation rather than temporary hardcoded values.

---

## 2. Why Hardcoded Thresholds Are Temporary

Hardcoded thresholds are acceptable for an MVP because they let the project demonstrate the full end-to-end pipeline quickly:

```text
sensor value breach
  -> Kafka anomaly alert
  -> KG + RAG + LLM diagnosis
  -> frontend diagnostic display
```

But in a real industrial system, hardcoded thresholds are not sufficient because thresholds are not universal.

For example:

```text
Temperature threshold for a compressor motor
≠ temperature threshold for a control panel
≠ temperature threshold for a pump bearing
≠ ambient room temperature threshold
```

Similarly:

```text
Vibration threshold for rotating equipment
≠ vibration threshold for static electrical equipment
```

and:

```text
Pressure threshold for one pipeline
≠ pressure threshold for another line with different rating
```

So the correct anomaly detector should be **device-aware** and **manual-grounded**.

Instead of asking:

```text
Is temperature greater than 70?
```

the future system should ask:

```text
Which device is this temperature sensor monitoring?
Which room/zone contains that device?
What does the device manual say is the safe operating temperature?
What warning and critical bands should be derived from that specification?
Is the current reading outside the allowed range for this device and condition?
```

That makes the anomaly detector engineering-valid rather than just demo-valid.

---

## 3. Target Architecture

The future design should introduce a **Manual-Derived Threshold Resolution Layer** between live sensor readings and anomaly decisions.

The upgraded flow should look like this:

```text
Device Manuals
  -> manual ingestion / parsing
  -> extract operating limits, safety limits, alarm limits
  -> store threshold specs with source references

Knowledge Graph
  -> maps rooms/zones to devices
  -> maps devices to sensors
  -> maps devices to manuals
  -> maps devices to failure modes

Live Sensors
  -> Redis latest telemetry
  -> threshold resolver queries KG + threshold store
  -> dynamic threshold chosen per device/sensor/zone
  -> anomaly decision made
  -> Kafka alert published if breached
  -> Neo4j + ChromaDB + LLM diagnostic pipeline
```

```text
                         ┌─────────────────────────┐
                         │     Device Manuals      │
                         │ PDF / DOCX / datasheets │
                         └────────────┬────────────┘
                                      │
                                      v
                         ┌─────────────────────────┐
                         │ Manual Extraction Layer │
                         │ limits, units, ranges   │
                         │ alarms, safety values   │
                         └────────────┬────────────┘
                                      │
                                      v
                         ┌─────────────────────────┐
                         │ Threshold Registry      │
                         │ device-specific limits  │
                         │ source + confidence     │
                         └────────────┬────────────┘
                                      │
                                      v
┌──────────────┐       ┌─────────────────────────┐       ┌────────────────────┐
│ Live Sensors │──────▶│ Threshold Resolver      │◀──────│ Neo4j Knowledge    │
│ Redis latest │       │ device + room + sensor  │       │ Graph              │
└──────────────┘       └────────────┬────────────┘       └────────────────────┘
                                    │
                                    v
                         ┌─────────────────────────┐
                         │ Anomaly Decision Engine │
                         │ normal / high / critical│
                         └────────────┬────────────┘
                                      │
                                      v
                         ┌─────────────────────────┐
                         │ Kafka: rigvision_alerts │
                         └────────────┬────────────┘
                                      │
                                      v
                         ┌─────────────────────────┐
                         │ KG + RAG + LLM Diagnosis│
                         └────────────┬────────────┘
                                      │
                                      v
                         ┌─────────────────────────┐
                         │ Frontend Diagnostics UI │
                         └─────────────────────────┘
```

---

## 4. Main Design Shift

The current design is:

```text
Zone -> sensor threshold
```

Example:

```text
zone_a temperature critical = 70
```

The future design should become:

```text
Zone -> devices in zone -> device manuals -> sensor-specific thresholds
```

Example:

```text
zone_a
  contains Compressor_A
  Compressor_A uses Manual_X
  Manual_X says:
      max operating temperature = 65°C
      alarm temperature = 70°C
      shutdown temperature = 80°C

sensor reading:
  temperature = 74°C

decision:
  above alarm threshold
  below shutdown threshold
  severity = HIGH
```

This is a much better model because the threshold is no longer arbitrary. It is tied to:

```text
actual device
actual manual
actual sensor type
actual zone
actual engineering source
```

---

## 5. Proposed Knowledge Graph Model

The knowledge graph should become the central mapping layer. It should know:
* Which rooms/zones exist?
* Which devices are installed in each zone?
* Which sensors are attached to or monitoring each device?
* Which manuals belong to each device?
* Which thresholds are defined by those manuals?
* Which failure modes are associated with threshold breaches?

### Nodes
* `(:Zone)`: id, rig_zone_id, name, floor, location_type
* `(:Device)`: id, name, type, manufacturer, model, serial_number, equipment_class
* `(:Sensor)`: id, type, unit, source, location, attached_to_device_id
* `(:Manual)`: id, title, manufacturer, model, document_type, version, source_path
* `(:ThresholdSpec)`: id, sensor_type, metric, normal_min, normal_max, warning_min, warning_max, critical_min, critical_max, unit, condition, operating_mode, source_text, page_number, confidence_score
* `(:FailureMode)`: id, name, description
* `(:Action)`: id, name, instruction

### Relationships
* `(:Zone)-[:CONTAINS]->(:Device)`
* `(:Device)-[:HAS_SENSOR]->(:Sensor)`
* `(:Device)-[:HAS_MANUAL]->(:Manual)`
* `(:Manual)-[:DEFINES_THRESHOLD]->(:ThresholdSpec)`
* `(:Device)-[:HAS_THRESHOLD]->(:ThresholdSpec)`
* `(:Device)-[:CAN_EXPERIENCE]->(:FailureMode)`
* `(:FailureMode)-[:INDICATED_BY]->(:ThresholdSpec)`
* `(:FailureMode)-[:REQUIRES_ACTION]->(:Action)`

This allows the detector to resolve thresholds dynamically.

---

## 6. Manual-Derived Threshold Extraction

The system should include a manual ingestion pipeline. Its job is to read device manuals and extract useful engineering limits.

Manuals may contain data like:
* Operating temperature: 0°C to 55°C
* Maximum winding temperature: 90°C
* Vibration alarm: 4.5 mm/s RMS
* Trip vibration: 7.1 mm/s RMS
* Maximum discharge pressure: 12 bar
* H2S alarm level: 10 ppm
* Shutdown level: 15 ppm
* Noise level: 85 dB

The extraction layer should convert this into structured threshold records.

Example structured output:

```json
{
  "device_model": "ACME-COMP-2200",
  "sensor_type": "temperature",
  "metric": "motor_winding_temperature",
  "unit": "C",
  "normal_min": 0,
  "normal_max": 55,
  "warning_min": 55,
  "warning_max": 75,
  "critical_min": 75,
  "critical_max": 90,
  "condition": "continuous operation",
  "source_document": "ACME-COMP-2200 Manual",
  "page_number": 42,
  "source_text": "Maximum continuous operating temperature shall not exceed 55°C...",
  "confidence_score": 0.91
}
```

This threshold should then be stored in the threshold registry and linked to the correct device/manual in Neo4j.

---

## 7. Threshold Registry

A threshold registry is needed because Neo4j is excellent for relationships, but threshold lookup also needs structured numeric values. You can store thresholds in Neo4j itself, or use a separate structured store such as Postgres/JSON files. For your current project scale, Neo4j alone is acceptable if you model `ThresholdSpec` nodes properly.

A threshold record should include more than just warning and critical values.

Recommended threshold schema:

```json
{
  "threshold_id": "thr_compressor_a_temperature_001",
  "device_id": "compressor_a",
  "device_model": "ACME-COMP-2200",
  "sensor_type": "temperature",
  "metric": "bearing_temperature",
  "unit": "C",
  "normal_range": {
    "min": 20,
    "max": 55
  },
  "warning_range": {
    "min": 55,
    "max": 70
  },
  "critical_range": {
    "min": 70,
    "max": 90
  },
  "operating_mode": "normal_load",
  "source": {
    "manual_id": "manual_acme_comp_2200",
    "page": 42,
    "section": "Operating Limits",
    "text": "Bearing temperature shall not exceed..."
  },
  "confidence": 0.91,
  "validated_by_human": false
}
```

The field `validated_by_human` is important. Since manuals can be complex, extracted thresholds should ideally be reviewed once before being trusted in a safety-critical context.

---

## 8. Runtime Threshold Resolution

At runtime, when `/api/diagnostics/run` is called, the backend should no longer directly use the hardcoded threshold values from `zone_definitions.json`.

Instead, the backend should use a `ThresholdResolver`.

The resolver’s job:
1. Find devices in this zone using Neo4j.
2. Find sensors attached to those devices.
3. Find threshold specs linked to those devices/manuals.
4. Filter threshold specs by sensor_type.
5. Normalize units.
6. Pick the most specific applicable threshold.
7. Return normal/warning/critical limits.

Conceptual function:

```python
def resolve_threshold(zone_id, sensor_type, sensor_id=None):
    devices = kg.get_devices_in_zone(zone_id)
    candidate_thresholds = []

    for device in devices:
        thresholds = kg.get_thresholds_for_device(
            device_id=device.id,
            sensor_type=sensor_type
        )
        candidate_thresholds.extend(thresholds)

    selected = choose_best_threshold(
        candidates=candidate_thresholds,
        sensor_id=sensor_id,
        operating_mode="normal_load"
    )

    return selected
```

Then the anomaly decision uses the resolved threshold:

```python
threshold = resolve_threshold(zone_id, sensor_type)

if value >= threshold.critical_min:
    severity = "CRITICAL"
elif value >= threshold.warning_min:
    severity = "HIGH"
else:
    severity = "NORMAL"
```

---

## 9. Threshold Selection Priority

Because multiple thresholds may exist, the system needs a clear priority order.

Recommended priority:
1. Exact sensor-specific threshold
2. Exact device model threshold
3. Device type/class threshold
4. Manufacturer generic threshold
5. Zone-level environmental threshold
6. Temporary fallback threshold from `zone_definitions.json`

Example:
* **Sensor**: temperature
* **Zone**: zone_a
* **Devices in zone**: `Compressor_A`, `Exhaust_Fan_A`, `Control_Panel_A`

The resolver should ask:
1. Is this temperature sensor attached to Compressor_A?
   * *yes* -> use Compressor_A manual threshold
2. If not attached to a specific device:
   * is it an ambient room temperature sensor?
     * *yes* -> use zone environmental threshold
3. If no manual threshold exists:
   * use temporary fallback threshold

This prevents the system from wrongly applying a compressor threshold to the whole room or a room ambient threshold to a motor winding.

---

## 10. Handling Multiple Devices in One Room

One room may contain several devices.
* **room_1** contains: compressor, pump, exhaust fan, gas detector, electrical panel

If a sensor is device-mounted, the logic is simple:
* `temperature_sensor_01` attached to `compressor` -> use compressor temperature threshold

But if the sensor is zone-level, such as an ambient temperature or gas sensor, it may monitor the room as a whole.

In that case, the system should use a different threshold type:
* ambient_temperature threshold
* gas_h2s room safety threshold
* noise exposure threshold

So the system must distinguish between **device-level sensors** and **zone-level environmental sensors**.

| Sensor | Scope | Threshold source |
| :--- | :--- | :--- |
| Motor winding temperature | Device-level | Motor/compressor manual |
| Bearing vibration | Device-level | Rotating equipment manual |
| Room temperature | Zone-level | room/environment safety policy |
| H2S gas sensor | Zone-level | safety standard/manual |
| Noise level | Zone-level | safety/environment limit |
| Pipeline pressure | Device/line-level | pipeline/pump/manual rating |

This distinction is critical.

---

## 11. Revised Real-Time Ingest & Diagnostic Flow

```text
SensorConsole
  -> POST /api/sensors
  -> Redis: rigvision:sensors:latest

RUN DIAGNOSTICS
  -> POST /api/diagnostics/run
  -> load latest sensor values
  -> for each zone:
       query Neo4j:
         zone -> devices -> sensors -> manuals -> thresholds
       resolve best threshold for each sensor
       compare reading against threshold
  -> if all normal:
       return all_clear
  -> if breached:
       publish manual-grounded alert to Kafka

Kafka: rigvision_alerts
  -> anomaly_listener.py
  -> graph_extractor.py gets failure modes/actions
  -> diagnostic_agent.py retrieves manuals from ChromaDB
  -> local LLM produces diagnostic report

Kafka: rigvision_diagnostics
  -> backend Kafka consumer
  -> Redis: rigvision:diagnostics
  -> WebSocket
  -> DiagnosticsModal
```

---

## 12. Future Alert Payload Should Include Threshold Source

The current alert lacks explaining metadata. The future alert should include **why** the system considered that sensor abnormal:

```json
{
  "event_id": "anom_1710000000_zone_a",
  "zone_id": "room_1",
  "rig_zone_id": "zone_a",
  "severity": "CRITICAL",
  "triggered_sensors": ["temperature"],
  "telemetry_snapshot": {
    "temperature": 72.0
  },
  "threshold_context": {
    "temperature": {
      "value": 72.0,
      "unit": "C",
      "normal_max": 55,
      "warning_min": 55,
      "critical_min": 70,
      "selected_threshold_id": "thr_compressor_a_temp_001",
      "device_id": "compressor_a",
      "device_name": "Compressor A",
      "source_manual": "Compressor A Operation Manual",
      "source_page": 42,
      "selection_reason": "Exact device-model threshold matched for Compressor A temperature sensor"
    }
  },
  "timestamp": 1710000000000
}
```

This makes the alert explainable. Now the LLM can reason with much better context:
* The temperature is not just high generally.
* It is above the critical threshold defined by the compressor manual for this specific device.

---

## 13. Updated Role of ChromaDB

Currently, ChromaDB is used mainly during diagnosis. Future design can use ChromaDB in two places:
1. **Offline threshold extraction:** Manuals are ingested into ChromaDB. The extraction pipeline retrieves relevant chunks (operating temp, alarms, vibration trip limits) to aid the LLM/extraction script in converting them to structured `ThresholdSpec` records.
2. **Runtime diagnosis:** Once an anomaly is detected, ChromaDB retrieves relevant manual sections to help the LLM explain (possible cause, maintenance action, inspection checklist).

The key rule: **Do not ask the LLM live: "What is the threshold?"** Instead, use the LLM/manual parser offline to extract thresholds, validate/store them, and perform deterministic lookup at runtime. This keeps runtime detection fast, auditable, and stable.

---

## 14. Updated Role of Neo4j

Neo4j is the system's **equipment-context resolver**.

At detection time, Neo4j answers:
* Which devices are in this room?
* Which sensors belong to those devices?
* Which manual applies to each device?
* Which thresholds apply to this sensor?

### Dynamic Threshold Spec Query
```cypher
MATCH (z:Zone {id: $zone_id})-[:CONTAINS]->(d:Device)
OPTIONAL MATCH (d)-[:HAS_SENSOR]->(s:Sensor {type: $sensor_type})
OPTIONAL MATCH (d)-[:HAS_THRESHOLD]->(t:ThresholdSpec {sensor_type: $sensor_type})
OPTIONAL MATCH (t)-[:SOURCED_FROM]->(m:Manual)
RETURN d, s, t, m
```

### Downstream Diagnostic Context Query
```cypher
MATCH (z:Zone {id: $zone_id})-[:CONTAINS]->(d:Device)
MATCH (d)-[:CAN_EXPERIENCE]->(f:FailureMode)
MATCH (f)-[:INDICATED_BY]->(symptom:Symptom)
WHERE symptom.type IN $triggered_sensors
MATCH (f)-[:REQUIRES_ACTION]->(a:Action)
RETURN d, f, collect(symptom), collect(a)
```

---

## 15. Operational Summary Diagram

```text
Manuals define what is safe.
  └── Neo4j maps which device is where.
        └── Redis tells what is happening now.
              └── Backend decides if it is abnormal.
                    └── Kafka sends the anomaly event.
                          └── Neo4j + ChromaDB + LLM explain the cause.
                                └── Frontend shows the result.
```
