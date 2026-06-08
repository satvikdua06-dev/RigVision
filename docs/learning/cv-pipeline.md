# CV Pipeline Walkthrough

Main file: `cv/pipeline.py`

This is the main runtime for the computer vision subsystem. It can run in three modes:

- `demo`: generates fake people and fake zone telemetry.
- `live`: reads real camera streams.
- `video`: reads video files and loops them for repeatable demos.

The pipeline writes live state into Redis for the backend and frontend.

## Top-Level Setup

The file imports standard libraries for arguments, base64 encoding, JSON, math, OS paths, queues, signals, threading, and time.

It imports NumPy for numeric operations and Redis for state output.

It sets OpenCV environment variables:

- `OPENCV_LOG_LEVEL=OFF` reduces noisy logs.
- `OPENCV_FFMPEG_CAPTURE_OPTIONS=rtsp_transport;tcp` makes RTSP streams more stable.

It inserts `cv/` into `sys.path` so imports like `tracking.triangulation` and `detection.detector` work when running `python pipeline.py` from inside `cv/`.

Why not package everything properly yet:

- For an internship/demo codebase, direct script execution is convenient.
- A future cleanup could convert `cv` into a real installable package.

## Global Events

`shutdown_event`

- Set when Ctrl+C arrives.
- Long-running loops check it to stop gracefully.

`clear_tracking_cache_event`

- Set when backend publishes the `clear_cache` command.
- Live/video loops use it to reset trackers and identity maps.

Why `threading.Event`:

- The pipeline uses threads for camera capture and Redis command listening.
- Events are a safe shared flag between threads.

## Redis Command Listener

Function: `redis_command_listener(redis_client)`

Workflow:

1. Subscribes to `rigvision:commands`.
2. Polls Redis pub/sub.
3. If message is `clear_cache`, sets `clear_tracking_cache_event`.
4. Sleeps briefly between polls.

Why pub/sub:

- Backend can send commands without knowing CV process internals.
- The CV process remains a separate service.

## Zone Loading

Function: `load_zone_definitions(path)`

- Opens `cad/zone_definitions.json`.
- Returns parsed JSON.

This is used when generating zone states, because zone definitions include max occupancy and sensor threshold metadata.

## Camera Frame Upload

Function: `create_redis_uploader(redis_client, upload_queue)`

Workflow:

1. Starts a daemon thread.
2. Pulls `(camera_id, frame)` items from a queue.
3. JPEG-encodes each frame.
4. Base64-encodes the JPEG.
5. Writes it to `rigvision:camera:frame:<camera_id>` with a short TTL.

Why a separate uploader thread:

- Detection/tracking loops should not block on JPEG encoding and Redis writes.
- If the frontend is not watching, the newest frame is still available.

Function: `_queue_latest_frame(upload_queue, cam_id, frame)`

- Tries to enqueue the frame.
- If the queue is full, drops one old item and tries again.

Why drop old frames:

- For a live camera preview, the latest frame matters more than every frame.
- Queueing stale frames causes visible lag.

Function: `annotate_and_enqueue(...)`

Workflow:

1. Copies the raw frame.
2. Draws bounding boxes around tracked people.
3. Chooses green/red color based on PPE hardhat status.
4. Builds a label from personnel ID or local track ID.
5. Draws posture skeleton if keypoints are present.
6. Sends annotated frame to upload queue.
7. Optionally displays OpenCV preview window.

Why annotate before upload:

- The frontend camera panel can show evidence overlays without running CV in the browser.

## Identity And Observation Merging

Function: `propagate_recognition(...)`

This spreads a recognized personnel ID across:

- global cross-camera IDs,
- local per-camera track IDs,
- recognition method labels.

Workflow:

1. For each matched cross-camera person, look for a track with `face_id`.
2. If found, attach that personnel ID to the global ID and all local track IDs.
3. If the global ID was already known, apply it to all local tracks.
4. If one local track was already known, promote that identity to the global ID.

Why this exists:

- Face/QR recognition may happen in only one camera.
- Once one camera recognizes a person, all matched views should share that ID.

Function: `build_fused_persons(...)`

This converts `MatchedPerson` objects into Redis-ready dictionaries.

For each person:

- Skip if no 3D position exists.
- Pick the best visible track by confidence.
- Use recognized personnel ID if available, otherwise global tracker ID.
- Round X/Y/Z coordinates.
- Assign zone/floor/posture/PPE/confidence/camera IDs.
- Merge duplicate observations with `_merge_person_observation`.

Function: `_merge_person_observation(existing, candidate)`

This handles two observations collapsing to the same person ID.

Rules:

- Prefer the observation with more camera visibility.
- If visibility ties, prefer higher confidence.
- Preserve recognition method if available.
- Merge and sort `camera_ids`.
- Recompute `cameras_visible`.

Why not simply overwrite:

- A lower-confidence camera view should not erase a better pose/zone estimate.
- Camera visibility must represent all cameras seeing the person, not only the chosen observation.

## Drawing Skeletons

Function: `_draw_skeleton(frame, keypoints)`

- Draws lines between common body keypoint pairs.
- Draws dots for each visible keypoint.

Why skeletons:

- Helps explain posture decisions visually.
- Provides evidence for sitting/bending/lying classifications.

## Stale Track Eviction

Function: `_evict_stale_tracks(...)`

This clears old identity mappings.

It removes:

- local track mappings no longer visible,
- global mappings no longer active,
- stale cross-camera mapper matches.

Why eviction matters:

- Track IDs can be reused by trackers.
- Stale identity maps can assign the wrong person to a new track.

## Zone State Generation

Function: `_generate_zone_states_from_persons(persons, zone_defs)`

Workflow:

1. For each zone in `cad/zone_definitions.json`, collect people currently in that zone.
2. Build PPE violation messages.
3. Set default status to warning if PPE violations exist, otherwise normal.
4. Check max occupancy.
5. Emit a zone state object with telemetry defaults, person count, PPE violations, and timestamp.

Why CV writes zone states:

- The dashboard should work even without sensor simulator running.
- The CV pipeline knows person counts and PPE summaries.

Why this can conflict with sensor bridge:

- Sensor bridge also writes `rigvision:zones`.
- A production version should merge sensor and CV state more carefully or split keys.

## Shared Loop Helpers

Function: `_normalize_floor_map(floor_map, source_count)`

- Ensures every camera/video source has a floor number.
- Missing entries default to floor 0.

Function: `_track_frames(frames, detector, trackers)`

- Sorts source IDs.
- Runs detector batch inference.
- Updates each source tracker with its matching detection list.

Why batch detection:

- YOLO can process multiple frames together more efficiently than one by one.

Function: `_write_realtime_state(redis_client, persons, zone_defs)`

- Generates zones.
- Writes `rigvision:persons`.
- Writes `rigvision:zones`.

Why centralize:

- Live and video mode previously repeated this exact Redis-write block.

Function: `_publish_annotated_frames(...)`

- Calls `annotate_and_enqueue` for every active source.
- Handles live mode and video mode with one helper.

Function: `_finish_frame(...)`

- Increments frame count.
- Every 30 frames, calculates FPS and writes it to Redis.
- Applies `--max-fps` throttling.

Function: `_shutdown_uploader(...)`

- Sends sentinel `None` to uploader thread.
- Joins it with timeout.

## Demo Mode

Class: `DemoDataGenerator`

Constructor:

- Loads zone definitions.
- Creates fake person objects with motion phases, speeds, PPE booleans, and posture.

`generate_persons()`

- Uses sine waves to move people through the 10m x 5m layout.
- Alternates floor by person ID.
- Clamps coordinates inside room bounds.
- Assigns zone using triangulation helper `assign_zone`.
- Randomly toggles PPE/posture occasionally.
- Emits Redis-ready person dictionaries.

`generate_zone_states(persons)`

- Calculates fake sensor readings using sine waves and random noise.
- Applies sensor warning/critical thresholds from zone definitions.
- Adds PPE and occupancy status.

Function: `run_demo_mode(...)`

- Runs generator at about 10 Hz.
- Writes fake persons and zones to Redis.

Why demo mode:

- Lets the frontend/backend be developed without cameras, GPU, or YOLO model.
- Great for presentations and testing UI interactions.

## Live Mode

Class: `ThreadedCamera`

Constructor:

- Opens USB camera index or RTSP/video source.
- Sets low buffer size.
- Starts a background thread that continuously reads frames.

Why threaded capture:

- OpenCV `VideoCapture.read()` can block.
- Detection loop should always grab the latest frame, not wait on slow capture.

Function: `run_live_mode(...)`

Workflow:

1. Normalize floor map.
2. Import OpenCV and heavy CV modules only when live mode starts.
3. Create detector.
4. Open threaded cameras.
5. Load calibrations; create defaults if missing.
6. Create per-camera trackers.
7. Create cross-camera mapper.
8. Load zone definitions for triangulation and zone states.
9. Start Redis uploader thread.
10. Loop:
    - clear caches if commanded,
    - read latest frames,
    - undistort and resize,
    - run batched detection and per-camera tracking,
    - cross-camera match,
    - triangulate 3D positions,
    - propagate recognized identity,
    - build Redis persons,
    - write Redis state,
    - publish annotated frames,
    - evict stale tracks,
    - update FPS and throttle.

Why calibrations:

- Multi-camera triangulation needs camera intrinsics and extrinsics.
- Without calibration, fallback defaults allow demo operation but reduce accuracy.

Why triangulation after matching:

- A 3D point needs observations of the same person from multiple camera views.
- Matching first decides which 2D detections belong together.

## Video Mode

Function: `run_video_mode(...)`

Video mode is similar to live mode but uses files instead of cameras.

Differences:

- Opens each path with `cv2.VideoCapture`.
- Loops videos by resetting frame position at EOF.
- Maps image foot point directly into zone bounds.
- Uses HSV histogram matching to associate tracks across videos.
- Uses per-video global ID offsets to reduce collisions.

Why use histogram matching here:

- Video demos may not have calibrated cameras.
- Simple color histograms provide a rough visual identity match.
- It is easier to explain and debug than full ReID for offline demo footage.

Current tradeoff:

- Histogram matching is weaker than learned ReID.
- Lighting, uniforms, and similar clothing can confuse it.

## CLI Entry Point

Function: `main()`

Defines arguments:

- `--mode`: `demo`, `live`, or `video`.
- `--cameras`: camera indexes, RTSP URLs, or video paths.
- `--confidence`: detection threshold.
- `--model`: YOLO model path.
- `--show-preview`: OpenCV preview windows.
- `--redis-host`, `--redis-port`, `--redis-password`.
- `--device`: model device, usually CUDA if available.
- `--resize-width`: resize frames before detection.
- `--max-fps`: throttle processing loop.
- `--floor-map`: maps each source to floor index.

Then it:

1. Resolves paths.
2. Connects to Redis.
3. Starts Redis command listener.
4. Dispatches to the selected mode.

Why CLI:

- Same script supports laptop demo, recorded videos, and live rig camera input.

