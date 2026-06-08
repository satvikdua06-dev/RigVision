# Knowledge And Diagnostics Walkthrough

This document explains the knowledge graph, RAG ingestion, anomaly listener, and diagnostic agent.

## `knowledge/graph/seed_graph.py`

Purpose: seed Neo4j with zones, devices, failure modes, symptoms, and actions.

Workflow:

1. Connect to Neo4j using environment variables.
2. Delete existing graph.
3. Create zones.
4. Create devices.
5. Link devices to zones with `CONTAINS`.
6. Create failure modes.
7. Link devices to failure modes with `CAN_EXPERIENCE`.
8. Create symptoms.
9. Link failure modes to symptoms with `INDICATED_BY`.
10. Create actions.
11. Link failure modes to actions with `REQUIRE_ACTION`.

Why a graph:

- Root-cause diagnosis is relational.
- A zone contains equipment; equipment can fail; failures show symptoms; failures require actions.
- Cypher can traverse those relationships cleanly.

Current caveat:

- The seed graph uses demo room IDs and household equipment. It should be aligned with `cad/zone_definitions.json` and ONGC rig equipment before relying on it for real diagnostics.

## `knowledge/extraction/query_generator.py`

Purpose: parse anomaly payloads and provide a safe Cypher query template.

`AnomalyQueryBuilder.process_payload(json_payload)`

Workflow:

1. Parse JSON string.
2. Extract `zone_id`.
3. Extract `triggered_sensors`.
4. Validate both exist.
5. Return query parameters.

Why validate early:

- Bad Kafka payloads should fail before hitting Neo4j or LLM calls.

`get_cypher_template()`

- Returns parameterized Cypher using `$zone_id` and `$sensor_types`.

Why parameterized queries:

- Avoids string-building Cypher from untrusted payloads.
- Keeps query shape stable and safer.

## `knowledge/extraction/graph_extractor.py`

Purpose: run Cypher against Neo4j and format graph context for the LLM.

`SubgraphExtractor.__init__`

- Reads `NEO4J_URI`, `NEO4J_USER`, and `NEO4J_PASSWORD`.
- Creates Neo4j driver.

`get_llm_context(parsed_zone, parsed_sensors)`

Workflow:

1. Find failure modes in the zone connected to triggered sensor symptoms.
2. Fetch all symptoms and actions for those failures.
3. Format each result into a compact text block.
4. Return a fallback message if no failure modes match.

Why two-step matching:

- Triggered sensors identify suspects.
- The LLM also needs non-triggered expected symptoms for negative reasoning.

Why text context:

- LLM prompts consume text, not graph records directly.
- Formatting controls what the model sees.

## `knowledge/agent_layer/rag_ingestion.py`

Purpose: load device manuals into ChromaDB.

Workflow:

1. Load environment variables.
2. Connect to ChromaDB HTTP server.
3. Read manual text file.
4. Split into chunks.
5. Generate Gemini embeddings.
6. Store chunks and embeddings in collection `device_manuals`.

Why ChromaDB:

- Manuals can be semantically searched by diagnostic context.
- The LLM prompt can include only relevant chunks instead of the entire manual.

Why embeddings:

- Keyword search may miss related wording.
- Embeddings retrieve by semantic similarity.

## `knowledge/agent_layer/diagnostic_agent.py`

Purpose: generate structured root-cause reports with Gemini.

`LLMDiagnosticAgent.__init__`

Workflow:

1. Load `GEMINI_API_KEY`.
2. Create Gemini client.
3. Connect to ChromaDB.
4. Load `device_manuals` collection.

Why fail if API key missing:

- Diagnostics cannot run without an LLM key.
- Early failure is clearer than silent empty reports.

`retrieve_manuals(graph_context)`

Workflow:

1. Embed graph context.
2. Query ChromaDB for top matching manual chunks.
3. Return joined text.

Why retrieve from graph context:

- The graph context already describes suspected devices/failures.
- That makes it a good semantic query.

`generate_report(telemetry, graph_context)`

Workflow:

1. Retrieve manual context.
2. Build prompt with telemetry, graph topology, and manual excerpts.
3. Ask Gemini for JSON.
4. Return response text.
5. On failure, return JSON with error.

Why strict JSON output:

- Backend and frontend can parse and render predictable fields.
- Free-form paragraphs are harder to display consistently.

## `knowledge/extraction/anomaly_listener.py`

Purpose: listen for anomaly alerts and publish diagnostic reports.

Workflow:

1. Create global `LLMDiagnosticAgent`.
2. Connect Kafka consumer to `rigvision_alerts`.
3. Connect Kafka producer to `rigvision_diagnostics`.
4. For each alert:
   - parse payload,
   - generate query params,
   - extract graph context,
   - call LLM diagnostic agent,
   - publish diagnostic JSON.

Why Kafka:

- Alert generation and diagnostic reasoning are separate services.
- LLM calls can be slow, so they should not block the sensor bridge or backend.

Current caveat:

- The module uses path hacks and local imports. Running it as a package may fail. A cleaner version should use package-relative imports or make `knowledge` an installable package.

## `knowledge/trigger.py`

Purpose: manually publish a sample anomaly alert.

Workflow:

1. Build hard-coded alert payload.
2. Connect Kafka producer.
3. Send to `rigvision_alerts`.
4. Flush and close.

Why this exists:

- Fast manual test of anomaly listener and backend diagnostics path.

Current caveat:

- The sample zone ID should match real RigVision zones.

