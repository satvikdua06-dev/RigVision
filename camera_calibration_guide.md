# RigVision-3D: Camera Calibration & 3D Reconstruction Guide

This guide details the physical setup, mathematical models, and implementation steps required to upgrade from a naive pixel-to-room mapping to a calibrated 2-camera 3D projection system.

---

## 1. Physical Setup & World Coordinates

To establish a unified spatial coordinate system, we must physically reference both cameras to a shared world origin.

### 1.1 Shared World Origin Definition
*   **Coordinate Convention:** We use a right-handed Cartesian coordinate system:
    *   **X-axis:** Along the length of the rig (0m to 10m).
    *   **Y-axis:** Vertically upward (0m to 6m, where Floor 0 is $Y \in [0, 3)$ and Floor 1 is $Y \in [3, 6]$).
    *   **Z-axis:** Along the width of the rig (0m to 5m).
*   **Origin ($[0,0,0]$):** Located on the bottom floor corner of Room A (inner corner, floor level).

```
          +Y (Up)
           ^
           |
           +---> +X (Length: Room A -> Corridor -> Room B)
          /
         v
       +Z (Width)
```

### 1.2 Camera Mounting & Coverage Optimization
*   **Mounting Positions:**
    *   **Camera 0 (Room A):** Mounted at $[0.5, 2.5, 0.5]$ (corner) looking downward at a pitch angle of $\sim 30^\circ$ and yaw of $\sim 45^\circ$ into Room A.
    *   **Camera 2 (Room B):** Mounted at $[9.5, 2.5, 4.5]$ looking back towards the corridor.
*   **Overlap Considerations:** To optimize for both stereo triangulation and full space tracking:
    *   Mount cameras high ($2.5\text{m}$ to $2.8\text{m}$) to minimize perspective occlusion and keep the ground plane fully visible.
    *   Create an overlap volume of at least $1.5\text{m} \times 2.0\text{m}$ in the corridor region to facilitate cross-camera ReID and multi-view triangulation.

### 1.3 Physical Measurements to Record
1.  **ArUco Marker Placements:** Tape 6x6 ArUco markers at stable, visible positions on the walls. Measure and record their exact centers in world meters (e.g., $[x, y, z]$ relative to the origin).
2.  **Floor Planes:** Record exact floor Y-levels (Floor 0: $Y = 0.05\text{m}$, Floor 1: $Y = 3.05\text{m}$).

---

## 2. Intrinsic Calibration

Intrinsic calibration computes lens parameters to correct radial/tangential distortion and obtain the camera intrinsic matrix $K$.

### 2.1 Checkerboard Setup
*   **Pattern:** A flat checkerboard pattern (e.g., $10 \times 7$ squares, yielding $9 \times 6$ inner corners).
*   **Physical square size:** Must be measured accurately (e.g., $25.0\text{mm}$).
*   **Captures:** Capture $15$ to $25$ frames. Ensure the checkerboard covers different areas of the lens (especially corners/edges) and is held at various angles and depths.

### 2.2 OpenCV Intrinsic Calibration Code
OpenCV finds chessboard corners, refines them, and computes $K$ and distortion coefficients:

```python
import numpy as np
import cv2
import json

def run_intrinsic_calibration(images, board_size=(9, 6), square_size_mm=25.0):
    # 3D points in real-world mm coordinates: (0,0,0), (25,0,0), ...
    objp = np.zeros((board_size[0] * board_size[1], 3), np.float32)
    objp[:, :2] = np.mgrid[0:board_size[0], 0:board_size[1]].T.reshape(-1, 2)
    objp *= square_size_mm

    obj_points = [] # 3D points in real world
    img_points = [] # 2D points in image plane

    for img in images:
        gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
        ret, corners = cv2.findChessboardCorners(gray, board_size, None)
        
        if ret:
            # Refine corner locations to sub-pixel accuracy
            criteria = (cv2.TERM_CRITERIA_EPS + cv2.TERM_CRITERIA_MAX_ITER, 30, 0.001)
            corners_refined = cv2.cornerSubPix(gray, corners, (11, 11), (-1, -1), criteria)
            
            obj_points.append(objp)
            img_points.append(corners_refined)

    # Calibrate
    h, w = images[0].shape[:2]
    ret, K, dist_coeffs, rvecs, tvecs = cv2.calibrateCamera(
        obj_points, img_points, (w, h), None, None
    )
    
    return ret, K, dist_coeffs
```

### 2.3 Storage and Format Matching
Calibrations are stored under `cv/calibration/configs/camera_X.json` in the format matching our `CameraCalibration` class:
```json
{
  "camera_source": "rtsp://...",
  "image_size": [640, 480],
  "camera_matrix": [
    [600.0, 0.0, 320.0],
    [0.0, 600.0, 240.0],
    [0.0, 0.0, 1.0]
  ],
  "dist_coeffs": [0.01, -0.02, 0.0, 0.0, 0.0],
  "rotation_matrix": [
    [1.0, 0.0, 0.0],
    [0.0, 1.0, 0.0],
    [0.0, 0.0, 1.0]
  ],
  "translation_vector": [[0.0], [0.0], [0.0]],
  "projection_matrix": [
    [600.0, 0.0, 320.0, 0.0],
    [0.0, 600.0, 240.0, 0.0],
    [0.0, 0.0, 1.0, 0.0]
  ]
}
```

---

## 3. Extrinsic & Coordinate Transform

Extrinsics define the camera’s rotation $R$ and translation $t$ in world space.

### 3.1 SolvePnP Workflow
1.  Place 4+ ArUco markers at known world coordinates $[X_w, Y_w, Z_w]$.
2.  Detect markers in the camera frame to find their 2D pixel centers $[u, v]$.
3.  Execute `cv2.solvePnP` passing the 3D-2D points, $K$, and distortion coefficients:
    ```python
    success, rvec, tvec = cv2.solvePnP(object_points, image_points, K, dist_coeffs)
    R, _ = cv2.Rodrigues(rvec)  # Convert rotation vector to 3x3 matrix
    ```

---

### 3.2 Transform Architectures: Homography vs. Triangulation

We evaluate both mapping approaches for our rig setup:

| Feature | (a) Ground-Plane Intersection / Homography | (b) Multi-View Triangulation (DLT) |
| :--- | :--- | :--- |
| **Mathematical Basis** | Ray-to-plane intersection: $Y_{\text{world}} = H$ | Direct Linear Transform (DLT) from 2+ views |
| **View Requirements** | Single-camera view (1 camera is sufficient) | Overlapping views (minimum 2 calibrated cameras) |
| **Output Type** | Establishes coordinates assuming ground level | Full $(X, Y, Z)$ 3D coordinates |
| **Real-world Accuracy** | $\approx 10\text{cm} - 30\text{cm}$ (highly dependent on tilt/dist) | $\approx 2\text{cm} - 8\text{cm}$ inside the overlap volume |
| **Stairs/Upper Decks** | Fails (assumes constant height floor) | Works perfectly |

### Which Fits Our Case?
**We must use a Hybrid Approach.**
Since our cameras cover separate rooms with limited overlap (Room A, Corridor, Room B), a pure stereo triangulation approach causes targets to vanish when they walk out of overlapping fields of view. However, a pure ground-plane intersection model fails to capture vertical climbs or multi-floor operations. 

Thus, our implementation employs:
*   **DLT Triangulation** as the primary tracker when a target is detected in multiple calibrated cameras.
*   **Ground-Plane Ray Intersection** as the fallback when the target is only seen by a single camera.

---

### 3.3 Implementation Code

Below is the production-ready implementation of both transform workflows:

```python
import numpy as np
import cv2

def ground_plane_intersection(pixel, K, R, t, floor_y=0.05):
    """
    Project a 2D pixel to a 3D point on a flat floor plane (Y = floor_y).
    """
    # 1. Convert pixel to normalized camera ray
    K_inv = np.linalg.inv(K)
    pixel_h = np.array([pixel[0], pixel[1], 1.0], dtype=np.float64)
    ray_cam = K_inv @ pixel_h
    
    # 2. Rotate ray into world space
    R_T = R.T
    ray_world = R_T @ ray_cam
    
    # 3. Calculate camera center in world coordinates
    cam_pos = (-R_T @ t).flatten()
    
    # 4. Find scalar lambda where: cam_pos.y + lambda * ray_world.y = floor_y
    if abs(ray_world[1]) < 1e-6:
        # Ray is parallel to ground plane
        return (float(cam_pos[0]), float(floor_y), float(cam_pos[2]))
        
    lam = (floor_y - cam_pos[1]) / ray_world[1]
    
    # 5. Compute intersection point
    position_3d = cam_pos + lam * ray_world
    return (float(position_3d[0]), float(floor_y), float(position_3d[2]))


def triangulate_dlt(pt1, pt2, P1, P2):
    """
    DLT Triangulation of a point seen in two calibrated cameras.
    """
    # Format points for OpenCV
    pts1 = np.array([[pt1[0]], [pt1[1]]], dtype=np.float64)
    pts2 = np.array([[pt2[0]], [pt2[1]]], dtype=np.float64)
    
    # Solve triangulation (returns homogeneous 4D coordinates)
    point_4d = cv2.triangulatePoints(P1, P2, pts1, pts2)
    
    # De-homogenize to 3D
    point_3d = point_4d[:3, 0] / point_4d[3, 0]
    return (float(point_3d[0]), float(point_3d[1]), float(point_3d[2]))
```

---

## 4. Failure Modes & Drift Management

1.  **Calibration Drift (Camera vibration/shift):**
    *   *Symptom:* Person locations shift over time; reprojection errors spike.
    *   *Mitigation:* Run online extrinsic recalibration periodically using static landmarks (e.g. fixed ArUco codes on walls) without needing a checkerboard.
2.  **Incorrect Height Assumptions:**
    *   *Symptom:* Person coordinates "jump" or offset horizontally when bending, jumping, or climbing.
    *   *Mitigation:* Apply Kalman filtering to smooth horizontal coordinates and check the camera's active floor index to update the target height plane.
3.  **Non-overlapping Blinds:**
    *   *Symptom:* Tracked IDs split or duplicate when transitioning between camera views.
    *   *Mitigation:* Use appearance ReID similarity embeddings (BoT-SORT) to match tracks across transitions.
