# CV Submodules Walkthrough

This document explains files under `cv/detection`, `cv/tracking`, and `cv/calibration`.

## `cv/detection/detector.py`

Purpose: turn raw image frames into person detections with optional PPE, posture, face ID, QR ID, and keypoints.

Key structures:

`Detection`

- `bbox`: pixel box `(x1, y1, x2, y2)`.
- `confidence`: model confidence.
- `class_id` and `class_name`: YOLO class metadata.
- `foot_point`: bottom-center of the box, used for floor projection.
- `ppe`: hardhat/vest/goggles state.
- `is_real_ppe`: whether PPE came from a real PPE model or fallback.
- `posture`: standing/sitting/bending/lying/unknown.
- `keypoints`: pose keypoints if RTMPose works.
- `face_id`, `face_confidence`, `recognition_method`: identity fields.

Why a dataclass:

- Detection is just structured data.
- A dataclass gives readable construction and default fields.

`PersonDetector`

Constructor responsibilities:

- Load YOLO model.
- Detect whether model is PPE-aware.
- Map class names to person/hardhat/vest/goggles.
- Try to initialize RTMPose.
- Fall back to rule-based posture if pose runtime is unavailable.

Why detect model capabilities:

- `yolov8l.pt` detects COCO people only.
- A custom PPE model can detect hardhats, vests, goggles.
- The same pipeline should run with both.

PPE association:

- PPE items are detected as separate boxes.
- `_associate_ppe_to_persons` checks how much each PPE box lies inside a person box.
- If containment passes threshold, that person gets that PPE item as present.

Why containment instead of nearest center:

- A hardhat box should be inside the person box.
- Nearest center can fail when people stand close together.

Posture classification:

- If pose keypoints exist, posture can use body geometry.
- If keypoints fail, fallback uses bounding-box aspect ratio.

Why fallback:

- Pose models are heavier and may fail on CPU or missing ONNX Runtime GPU.
- The dashboard still needs a posture value.

Face/QR recognition:

- The detector attempts identity recognition from the cropped person area.
- QR recognition is useful in controlled demos where people wear tags.

Why identity in detector:

- Recognition depends on raw image crops, before tracking loses pixel detail.

## `cv/tracking/tracker.py`

Purpose: keep stable per-camera person IDs across frames.

`TrackedPerson`

- Similar to `Detection`, but with `track_id`.
- Includes tracking counters and optional feature vector.
- `__post_init__` calculates bottom-center foot point from bounding box.

`compute_iou`

- Calculates intersection-over-union between two boxes.
- Used to associate BoT-SORT output back to the original detection metadata.

`PersonTracker`

Constructor:

- Creates default BoT-SORT arguments.
- Initializes a BoT-SORT tracker.

`update(frame, detections)`

Workflow:

1. Convert detections into BoT-SORT input array.
2. Run tracker update.
3. For each returned track, find best matching detection by IoU.
4. Copy PPE, posture, keypoints, face/QR identity onto `TrackedPerson`.
5. Return tracked people.

Why tracking:

- YOLO detections do not have persistent IDs.
- Compliance and UI need stable person numbers.

Why BoT-SORT:

- It combines motion prediction and assignment.
- It is stronger than plain IoU matching during occlusion or short misses.

## `cv/tracking/cross_camera.py`

Purpose: map per-camera track IDs into global person IDs.

`MatchedPerson`

- Holds one global ID.
- Stores per-camera tracks for the same person.
- Later receives 3D position and zone.

`compute_epipolar_distance`

- Measures whether two 2D points from different cameras are geometrically plausible.
- Uses the fundamental matrix when calibration exists.

Why epipolar geometry:

- The same 3D point must lie on corresponding epipolar lines across camera views.
- This reduces false matches between visually similar people.

`compute_appearance_distance`

- Uses color/appearance features to compare people.

Why appearance:

- Geometry alone can be ambiguous.
- Appearance alone can be fooled by similar uniforms.
- Combining both is better.

`CrossCameraMapper`

Responsibilities:

- Compare tracks across cameras.
- Create or reuse global IDs.
- Keep previous matches for continuity.
- Produce `MatchedPerson` objects.

Why previous matches:

- Cross-camera assignment should not flicker when confidence changes slightly.

## `cv/tracking/triangulation.py`

Purpose: convert 2D detections from camera views into 3D rig coordinates.

`CameraCalibration`

Stores:

- camera ID,
- intrinsic matrix `K`,
- distortion coefficients,
- rotation `R`,
- translation `t`,
- image size.

Why intrinsics and extrinsics:

- Intrinsics explain how a camera maps 3D rays into pixels.
- Extrinsics explain where the camera sits in world space.

`load_calibrations(configs_dir)`

- Reads calibration JSON files.
- Builds camera calibration objects.

`triangulate_dlt`

- Direct Linear Transform triangulation.
- Uses projection matrices from two cameras.

Why DLT:

- Standard method for triangulating 3D points from calibrated camera pairs.
- Works well when calibration is decent and both cameras see the person.

`ground_plane_intersection`

- Projects a single camera ray onto a floor plane.

Why fallback to ground plane:

- Sometimes only one camera sees a person.
- A rough floor projection is better than no position for the digital twin.

`load_zones`

- Converts `cad/zone_definitions.json` bounds into easy min/max tuples.

`assign_zone`

- Checks if `(x, y, z)` lies inside a zone bounding box.
- Returns zone ID or `unknown`.

`triangulate_all`

- Applies triangulation/fallback projection to every matched person.
- Assigns zone after computing position.

## Calibration Scripts

### `cv/calibration/calibrate_intrinsic.py`

Purpose: find camera intrinsics and distortion coefficients from checkerboard images.

Workflow:

1. Parse camera ID and image path pattern.
2. Find matching images.
3. Detect checkerboard corners.
4. Refine corners.
5. Run OpenCV camera calibration.
6. Save matrix and distortion coefficients to JSON.

Why checkerboard calibration:

- Checkerboard corners provide known 3D-to-2D correspondences.
- OpenCV has reliable built-in calibration routines.

### `cv/calibration/calibrate_extrinsic.py`

Purpose: estimate the relative pose between two cameras.

Workflow:

1. Load intrinsics for master and target cameras.
2. Load synchronized image pair.
3. Detect checkerboard in both.
4. Refine corners.
5. Run stereo calibration.
6. Save rotation, translation, and fundamental matrix.

Why synchronized images:

- Both cameras must see the checkerboard in the same physical pose.
- Otherwise relative pose calculation is invalid.

## BoT-SORT Files

These files implement the tracking algorithm internals:

- `basetrack.py`: base track states and IDs.
- `bot_sort.py`: main BoT-SORT tracker logic.
- `kalman_filter.py`: motion prediction.
- `matching.py`: assignment and distance utilities.
- `tracking_utils/io.py`: MOT result input/output helpers.
- `tracking_utils/evaluation.py`: evaluation helpers.
- `tracking_utils/timer.py`: simple timing utility.

Why keep them vendored:

- The project can run without installing a separate BoT-SORT package.
- It allows local fixes if needed.

Why avoid editing them casually:

- Tracker internals are algorithmic and easy to break.
- Most project-level changes should happen in `tracker.py`, which wraps these helpers.

