# RigVision-3D — Camera World-Pose Calibration Guide (AprilTag)

**Goal:** find each camera's **position and orientation in the room's world coordinate
frame** — the missing piece that lets us triangulate people *directly* into room
coordinates instead of faking it in `place_in_zone()`.

This is the one calibration step the project skipped. Once you have it, a tilted /
rotated phone is no longer a problem: the tilt is baked into the math and cancels out.

> **World frame reminder (from `cad/zone_definitions.json`):**
> `X = length, Y = up, Z = width`, units = **meters**, origin `(0,0,0)` at the
> ground-floor corner of Room A. The **floor is the X–Z plane at y = 0**.

---

## 0. TL;DR

| Question | Answer |
|---|---|
| AprilTag or ArUco? | **AprilTag (family `tag36h11`)** for the world-pose step. More robust + accurate at distance / poor light, far fewer false positives. (You can *detect* AprilTags through `cv2.aruco` too — see §1.) |
| How many markers? | **4 minimum, 6–8 recommended.** One tag = 4 corners but is *ambiguous*. More tags, spread out, kills the ambiguity and averages out noise. |
| Which points (locations)? | Tags at **surveyed, tape-measured room positions**: most flat on the floor spread across the camera's view, **plus 2 on the walls at a known height** to break coplanarity. Each corner's `(x,y,z)` in meters must be known. |
| What's the output? | Per camera: `R_world`, `t_world` (world→camera) → projection matrix `P = K[R|t]`. Saved as `world_pose_cam_{id}.npz`. |
| What does it replace? | The `place_in_zone()` slide + mirror + clamp hack. Triangulation then outputs world coords directly. |

---

## 1. AprilTag vs ArUco — the real decision

Both are square black-and-white fiducial markers. Both give you **4 corner points**
per marker with sub-pixel detection. The difference is in robustness and tooling.

| | **AprilTag** (`tag36h11`) | **ArUco** (`DICT_5X5`, etc.) | **ChArUco board** |
|---|---|---|---|
| Detection robustness (distance, blur, low light) | **Best** | Good | Good |
| False-positive rate | **Very low** | Higher | Low |
| Corner sub-pixel accuracy | High (native detector refines edges) | High | **Highest** (chessboard corners) |
| Extra dependency | `pupil-apriltags` (or use `cv2.aruco` AprilTag dicts) | none (built into OpenCV) | none (OpenCV) |
| Best use here | **Markers at surveyed room points** ← our case | quick/no-dep alternative | one rigid board at a known pose |

### Recommendation for RigVision

**Use AprilTag `tag36h11`.** Your environment is a fixed-camera industrial rig with
DroidCam phones mounted in corners looking *across* a bay — markers will be seen at an
angle, at a few meters' distance, possibly under uneven lighting. AprilTag's robustness
and low false-positive rate matter more here than the tiny corner-accuracy edge a
ChArUco board would give. It is the right call.

Two ways to actually run the detector:

1. **Native `pupil-apriltags`** (recommended for best corner accuracy):
   ```bash
   pip install pupil-apriltags
   ```
2. **Through OpenCV** (zero extra deps — `cv2.aruco` ships AprilTag dictionaries):
   ```python
   aruco_dict = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_APRILTAG_36h11)
   ```
   Slightly less refined corners than the native detector, but perfectly usable.

> **Why not just one big ChArUco board?** A board only gives you the pose of *that
> board's location*. With individual tags surveyed at spread-out, non-coplanar room
> points you get a much stronger, less ambiguous world-pose solve. If you'd rather use a
> board, place it flat with one corner exactly at a known room point and one edge aligned
> with the X axis — but tags are more flexible for a real rig.

---

## 2. What we are actually solving (concepts)

`cv2.solvePnP` answers exactly one question:

> *"I know these 3D points in the **world** (because I tape-measured them), and I see
> them at these **pixels**. Given the camera's lens (`K`, `dist`), how must the camera be
> **positioned and rotated** to produce that image?"*

It returns `rvec, tvec` where (with `R = cv2.Rodrigues(rvec)[0]`):

```
X_camera = R @ X_world + t          # this (R, t) is the WORLD→CAMERA transform
```

That `[R | t]` is **exactly the extrinsic matrix** you drop into the projection matrix:

```
P = K @ [R | t]                     # 3x4, the thing triangulation needs
```

Two derived quantities you'll want for sanity checks:

```
Camera CENTER in world:      C = -R.T @ t          # where the phone physically is
Camera ORIENTATION in world: R_cam = R.T           # columns = camera axes in world
```

The camera center `C` should match where you mounted the phone (compare to the
`position` values in `zone_definitions.json`). That's your first reality check.

---

## 3. Prerequisites (do these first)

1. **Intrinsics already calibrated** for every camera → `configs/intrinsics_cam_{id}.npz`
   (`K`, `dist`). World-pose calibration *depends* on these. Run
   `capture_intrinsic.py` + `calibrate_intrinsic.py` first if you haven't.
2. **Cameras in their FINAL mounted positions.** Do not move a phone after calibrating
   its world pose. (Same rule as extrinsic calibration.)
3. **Print AprilTags** at a known physical size. Measure the **black square side
   length** with calipers/ruler after printing — printers scale. Call it `TAG_SIZE_M`
   (e.g. `0.15` for 15 cm). Mount each tag on something **rigid and flat** (foam board /
   acrylic) so it doesn't curl. A curled tag wrecks corner accuracy.
4. **Give every tag a unique ID** from `tag36h11` (ID 0, 1, 2, …) and write down which
   ID goes where.

---

## 4. How many markers, and which points? (the crux of your question)

### How many

| # of tags | Corners | Verdict |
|---|---|---|
| 1 | 4 | **Minimum, but ambiguous.** A single planar square has a classic *two-solution* pose flip, especially when seen nearly head-on or far away. Avoid as your only tag. |
| 2–3 | 8–12 | Workable; ambiguity mostly resolved if they're spread out. |
| **4–6** | **16–24** | **Recommended.** Strong least-squares solve, noise averages out, robust to one bad detection. |
| 6–8 | 24–32 | Best. Diminishing returns beyond this. |

**Use ≥ 4 tags.** Each tag contributes 4 corners, and `solvePnP` is a least-squares fit
over *all* corners at once — more well-spread points = more accurate, more stable pose.

### Which points (this matters as much as the count)

Three rules:

1. **Spread them across the camera's field of view** — corners of the floor area, not
   clustered in the middle. Points near the image edges constrain the pose far better
   than points all bunched in the center.

2. **Break coplanarity.** If *all* tags lie flat on the floor (all at `y = 0`), they're
   coplanar, and pose accuracy along the viewing direction (depth) degrades and the flip
   ambiguity creeps back. Fix it by putting **at least 2 tags on the walls at a known
   height** (e.g. `y = 1.5 m`). Floor tags + wall tags = a 3D spread = a strong solve.

3. **Survey each tag precisely.** You must know each tag's **center `(x, y, z)` in room
   meters** *and* its **orientation** (which way is "up" / which edge is along X), because
   the 4 corner world-coordinates are derived from those. Measure from the room origin
   (Room A's corner) with a tape measure / laser distance meter to **±2–3 mm**. Your
   final accuracy can't beat your tape measurements.

### A concrete recommended layout (per camera / per room)

For Room A (zone_a), origin at its corner, floor = X–Z plane at y=0:

```
   Z (width) ↑
   3.85 ┌─────────────────────────────┐
        │  [T2 wall]          [T3 wall]│   ← 2 tags on the far/side walls, y≈1.5m
        │                              │
        │     ▣T4            ▣T5        │   ← floor tags spread to the corners
        │                              │
        │  ▣T0            ▣T1           │
      0 └─────────────────────────────┘→ X (length)
        0                            5
```

- `T0..T5` are AprilTag IDs.
- Floor tags (`T0,T1,T4,T5`) lie flat → all 4 corners at `y = 0`.
- Wall tags (`T2,T3`) are vertical → corners share a known `y` band.
- Place them so **both cameras of the zone can see as many as possible in their
  overlapping region** — that's what makes the next step (§7) clean.

> **Pro tip:** put the tags' centers at round, easy-to-measure spots
> (e.g. `x = 1.0, z = 1.0`) so surveying is less error-prone.

---

## 5. Defining each tag's corner world-coordinates

The detector returns the 4 image corners in a **fixed order**. For `pupil-apriltags`
the corner order is: `[bottom-left, bottom-right, top-right, top-left]` *in the tag's own
frame*. (For `cv2.aruco` it is `[top-left, top-right, bottom-right, bottom-left]`.)
**Your `objectPoints` (world) order MUST match the detector's `imagePoints` order** —
this is the #1 silent bug. Verify once by drawing the detected corners and the axis.

For a tag of side `s = TAG_SIZE_M`, half = `h = s/2`:

**Floor tag** (flat, lying in X–Z plane, center `(cx, 0, cz)`, tag's local +X along world
+X, local +Y along world +Z):
```python
# order must match your detector; this example matches pupil-apriltags (BL,BR,TR,TL)
obj = np.array([
    [cx - h, 0.0, cz - h],   # bottom-left
    [cx + h, 0.0, cz - h],   # bottom-right
    [cx + h, 0.0, cz + h],   # top-right
    [cx - h, 0.0, cz + h],   # top-left
], dtype=np.float64)
```

**Wall tag** (vertical, on the X–Y wall at `z = z0`, center `(cx, cy, z0)`):
```python
obj = np.array([
    [cx - h, cy - h, z0],
    [cx + h, cy - h, z0],
    [cx + h, cy + h, z0],
    [cx - h, cy + h, z0],
], dtype=np.float64)
```

Store this as a small JSON map `tag_id → 4 world corners` (your "survey file"), e.g.
`configs/world_tags_zone_a.json`. This file *is* your ground truth.

---

## 6. Step-by-step: capturing & solving (single camera)

### 6.1 Capture
1. Mount all tags at their surveyed positions. Don't move them or the camera.
2. Grab **one clean still** from the camera at the **same resolution the live pipeline
   uses** (reuse the `LatestFrame` reader from `capture_intrinsic.py`). One frame is
   enough, but grab a few and average / pick the sharpest.

### 6.2 Detect
```python
import cv2, numpy as np
from pupil_apriltags import Detector

K, dist = load_intrinsics("configs/intrinsics_cam_0.npz")   # your loader
gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)

det = Detector(families="tag36h11")
detections = det.detect(gray)        # each has .tag_id and .corners (4x2, pixel)
```

### 6.3 Build correspondences
```python
survey = load_json("configs/world_tags_zone_a.json")   # tag_id -> 4 world corners

obj_pts, img_pts = [], []
for d in detections:
    if str(d.tag_id) not in survey:
        continue                      # ignore tags we didn't survey
    obj_pts.extend(survey[str(d.tag_id)])     # 4 world points
    img_pts.extend(d.corners.tolist())        # 4 pixel points (SAME order!)

obj_pts = np.array(obj_pts, dtype=np.float64)
img_pts = np.array(img_pts, dtype=np.float64)
assert len(obj_pts) >= 8, "need >=2 tags; aim for >=4"
```

### 6.4 Solve (robust, no init needed)
```python
# SQPNP is a solid global PnP solver for many points; RANSAC guards against a bad tag.
ok, rvec, tvec, inliers = cv2.solvePnPRansac(
    obj_pts, img_pts, K, dist,
    reprojectionError=3.0, flags=cv2.SOLVEPNP_SQPNP)

# Polish with a Levenberg–Marquardt refinement over the inliers.
rvec, tvec = cv2.solvePnPRefineLM(
    obj_pts[inliers.ravel()], img_pts[inliers.ravel()], K, dist, rvec, tvec)

R, _ = cv2.Rodrigues(rvec)            # 3x3 world->camera rotation
t = tvec.reshape(3, 1)
P = K @ np.hstack([R, t])             # the projection matrix we wanted
```

### 6.5 Validate (do not skip)
```python
# (a) Reprojection error — should be ~1px or less.
proj, _ = cv2.projectPoints(obj_pts, rvec, tvec, K, dist)
err = np.linalg.norm(img_pts - proj.reshape(-1, 2), axis=1).mean()
print(f"reprojection error = {err:.3f}px")     # >2px => bad survey or wrong corner order

# (b) Camera center in world — compare to where you mounted the phone.
C = (-R.T @ t).ravel()
print(f"camera center (world) = {C}")          # should match zone_definitions cam position

# (c) Camera 'up'/'forward' sanity: the camera should be high (C[1] ~ 2.7m) and
#     looking down/across into the room, not underground or facing a wall.
```

### 6.6 Save
```python
np.savez_compressed("configs/world_pose_cam_0.npz",
                    R=R, t=t, P=P, rvec=rvec, tvec=tvec,
                    reprojection_error=err)
```

Repeat 6.1–6.6 for **every camera** (0,1,2,3). Cameras in the same zone should be solved
against the **same survey file**, so they all land in the same world frame.

---

## 7. Wiring it into the RigVision pipeline

Right now `triangulation.py` makes the master camera a fake origin (`R=I, t=0`) and the
target camera carries only the *camera-to-camera* `R,T`. Replace that with **both
cameras' world poses**:

```python
# Both P matrices are now in the WORLD frame:
P0 = K0 @ [R0 | t0]      # from world_pose_cam_0.npz
P1 = K1 @ [R1 | t1]      # from world_pose_cam_1.npz

pos_world = triangulate_dlt(foot_a, foot_b, P0, P1)   # ALREADY in room coordinates
```

Because both projection matrices are world-referenced, `triangulate_dlt` now returns a
point **directly in room meters** — correct depth, correct sideways position, camera tilt
fully accounted for.

**Consequence:** `place_in_zone()` collapses to almost nothing. You no longer add a
camera offset, no longer mirror-flip, no longer clamp to hide errors. The only thing you
might keep is a tiny `y = floor + 0.05` "stand on the floor" snap and a soft clamp as a
safety net.

> **Fallback chaining (if a camera can't see enough tags):** if only the master sees
> enough tags, derive the target from the stereo extrinsics you already calibrated:
> `R1 = R_01 @ R0`, `t1 = R_01 @ t0 + T_01`. But solving *each* camera independently
> against the same tags is more accurate and self-consistent — prefer that.

---

## 8. Common pitfalls (read before you start)

1. **Corner-order mismatch** between `objectPoints` and `imagePoints`. Different
   detectors order corners differently. Symptom: huge reprojection error or a pose that's
   rotated 90°/180°. Fix: draw both and confirm corner 0 maps to corner 0.
2. **Wrong tag size.** `TAG_SIZE_M` must be the *printed* black-square side, measured
   after printing. Wrong size = correct rotation but wrong distance/scale.
3. **All tags coplanar.** Floor-only layout → weak depth + flip ambiguity. Add wall tags.
4. **Sloppy survey.** Tape-measure error propagates 1:1 into pose error. Measure to mm,
   use round positions, double-check against the room origin.
5. **Curled / non-flat tags.** Mount on rigid board.
6. **Calibrating at the wrong resolution.** Detect and solve at the **same pixel width
   the live pipeline triangulates at**, or scale `K` accordingly (the pipeline already
   scales `K` to `triag_width` — keep it consistent).
7. **Distortion ignored.** Always pass `dist` to `solvePnP` and `projectPoints`. Don't
   pre-undistort *and* pass dist (you'd correct twice).
8. **Moving anything afterward.** Re-survey / re-solve if a camera or tag shifts.

---

## 9. Suggested file layout

```
cv/calibration/
├── capture_world_pose.py     # grab a still from a camera (reuse LatestFrame)
├── calibrate_world_pose.py   # detect tags -> solvePnP -> world_pose_cam_{id}.npz
├── configs/
│   ├── intrinsics_cam_{id}.npz       # (prereq, already exists)
│   ├── world_tags_zone_a.json        # survey: tag_id -> 4 world corners
│   ├── world_tags_zone_b.json
│   └── world_pose_cam_{id}.npz       # OUTPUT: R, t, P per camera
```

Then `load_zone_calibrations()` in `triangulation.py` loads `world_pose_cam_{id}.npz`
instead of fabricating the master-origin frame, and `place_in_zone()` becomes a thin
floor-snap/clamp.

---

## 10. One-paragraph summary

Print **≥ 4 AprilTags (`tag36h11`)**, mount them at **tape-measured room positions** —
most on the floor spread to the corners of each camera's view, **plus 2 on the walls** to
break coplanarity — and record each tag's 4 corner world-coordinates. For each camera,
detect the tags, match corners to their known world points, and run
`solvePnPRansac → solvePnPRefineLM` to get `R, t` (world→camera). Validate via
reprojection error (~1px) and by checking the recovered camera center matches where the
phone is mounted. Save `P = K[R|t]` per camera. Feed both cameras' world-frame `P`
matrices into `triangulate_dlt`, and it outputs room coordinates directly — no more
`place_in_zone` fudge, and any camera tilt is handled exactly.
```