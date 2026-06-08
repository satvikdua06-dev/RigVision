# Frontend Walkthrough

The frontend is a React app powered by Vite, Zustand, Three.js, and React Three Fiber.

Its main job is to turn Redis/WebSocket state into an interactive 3D digital twin and operator dashboard.

## `frontend/src/main.jsx`

Purpose: React entry point.

Workflow:

1. Import React and ReactDOM.
2. Import global CSS.
3. Import `App`.
4. Render `<App />` into the DOM element with ID `root`.

Why keep it tiny:

- App-level logic belongs in `App.jsx` and components.
- The entry file should only bootstrap React.

## `frontend/src/App.jsx`

Purpose: top-level layout and backend connection.

Workflow:

1. Pull `connectToBackend` from Zustand store.
2. On mount, call `connectToBackend()`.
3. Render vertical app layout:
   - `TopBar`
   - `Sidebar`
   - central `Scene3D`
   - `CameraFeeds` overlay
   - watermark/control hints
   - `DiagnosticsModal`

Why connect here:

- `App` is mounted once.
- The WebSocket connection should be global, not tied to a specific panel.

## `frontend/src/stores/useRigStore.js`

Purpose: global application state and backend communication.

State groups:

- Real-time data: `persons`, `zones`, `violations`, `diagnostics`.
- Connection state: `connected`, `_ws`, `_reconnectTimer`.
- UI selection: `selectedPerson`, `selectedZone`, `sidebarTab`.
- Visibility toggles: `showSensors`, `showAvatars`, `showDiagnosticsModal`.
- Scene controls: wall opacity, FPS limit, floor filter, zone select mode.
- Render decoupling internals: `_latestRawData`, `_rafId`, `_lastRenderTime`.

Why Zustand:

- Simple store API.
- No Redux boilerplate.
- Components can subscribe to specific slices.

WebSocket workflow:

1. Build WebSocket URL from environment or hostname.
2. Connect to `/ws/realtime`.
3. On `realtime_update`, store raw data in `_latestRawData`.
4. Render loop applies `_latestRawData` at the selected FPS limit.
5. On close, schedule reconnect after two seconds.

Why render decoupling:

- Backend can push data faster than the 3D scene should render.
- Decoupling prevents unnecessary React/Three work.

Control actions:

- `toggleVlmGating`: POSTs to backend setting endpoint.
- `clearTrackingCache`: POSTs to backend command endpoint.
- `selectPerson`, `selectZone`, `clearSelection`: manage inspector selection.
- `toggleSensors`, `toggleAvatars`: show/hide scene layers.

## `frontend/src/components/Scene3D.jsx`

Purpose: main Three.js digital twin.

Important inner components:

`RenderThrottler`

- Uses `useFrame` to manually render at selected FPS.
- Reduces GPU workload.

`RigRoom`

- Loads GLTF room model.
- Clones scene and materials so duplicated rooms do not share material mutation.
- Scales and centers room model into target zone footprint.
- Applies wall opacity.
- Handles zone/person click selection.
- Shows hover popup for model parts.

Why clone materials:

- Room A and Room B need independent opacity/highlight behavior.
- Shared materials would cause edits in one room to affect the other.

`CorridorBridge`

- Procedurally renders platform, support pillars, handrails, and safety glass.
- Handles corridor zone selection.

Why procedural corridor:

- Easier than finding/exporting a perfect CAD corridor asset.
- Fits exact 2m x 2m connector dimensions.

`Lighting`

- Defines ambient, directional, point, and hemisphere lights.

`Floor`

- Draws floor planes and grids for floor 0 and floor 1.

`SettingsPanel`

- UI overlay for floor filter, interaction target, wall opacity, FPS limit, VLM gating, and cache clear.

Main `Scene3D`

Workflow:

1. Subscribe to persons, zones, toggles, and floor filter.
2. Compute which floors are visible.
3. Render Canvas.
4. Render room/corridor geometry.
5. Render zone overlays from live zone data.
6. Render person avatars.
7. Render cameras and sensors.
8. Attach orbit controls and viewport gizmo.

Why React Three Fiber:

- Lets React state drive Three.js scene objects.
- Components map cleanly to 3D concepts.

## `frontend/src/components/ZonePlane.jsx`

Purpose: translucent zone status overlay.

Workflow:

1. Read selected zone, selection actions, interaction mode, and modal state.
2. Pick color based on zone status.
3. Use static zone definition for center and size.
4. Animate critical zones with pulsing opacity/emissive intensity.
5. On click, select zone unless an avatar was clicked.
6. Render floor tint, label, and selected-zone telemetry popup.

Why floor tint:

- It shows status without hiding the room model.

Why use static zone definition:

- Redis zone state contains telemetry/status, not physical bounds.

## `frontend/src/components/PersonAvatar.jsx`

Purpose: render a tracked person in 3D.

Workflow:

1. Read selected person and modal state.
2. Calculate color from PPE/posture.
3. Smoothly interpolate mesh position toward latest coordinates.
4. Pulse alert ring for PPE violations.
5. On click, select person.
6. Show popup with ID, zone, posture, confidence, PPE.

Why interpolate:

- CV updates can jump slightly.
- Smooth movement looks more like a live digital twin.

## `frontend/src/components/CameraFeeds.jsx`

Purpose: camera evidence panel for selected person.

Workflow:

1. If no person is selected, render nothing.
2. Determine camera IDs from `person.camera_ids`, fallback to 0/1/2.
3. Show selected person details and PPE status.
4. Render each MJPEG feed as an `<img>`.
5. If an image fails, show offline panel and retry later.

Why `<img>` for MJPEG:

- Browser handles multipart MJPEG streams naturally.
- No custom video decoder needed.

## `frontend/src/components/Sidebar.jsx`

Purpose: left operator panel.

Subcomponents:

`SensorBar`

- Displays telemetry value against warning/critical thresholds.

`ZonesTab`

- Searchable list of zones.
- Shows status, telemetry bars, person count, and selection state.

`PersonsTab`

- Searchable list of people.
- Shows zone, posture, confidence, camera visibility, and PPE chips.

`ViolationsTab`

- Lists compliance violations with severity and age.

Main `Sidebar`

- Manages tabs.
- Shows counts.
- Provides toggles for avatars and sensors.

Why sidebar:

- Operators need quick scanning and precise selection separate from the 3D scene.

## `frontend/src/components/TopBar.jsx`

Purpose: compact global status bar.

Shows:

- Critical zone count.
- Warning zone count.
- PPE alert person count.
- Critical violation count.
- Diagnostics button/count.
- Connection/time status.

Why top bar:

- Gives at-a-glance operational state even when the 3D scene is busy.

## `frontend/src/components/DiagnosticsModal.jsx`

Purpose: detailed incident/LLM diagnostic viewer.

Workflow:

1. Read diagnostics and modal visibility from store.
2. Auto-select first/latest diagnostic.
3. Render event list on left.
4. Render selected event details on right:
   - severity,
   - event ID,
   - zone,
   - diagnosis,
   - confidence,
   - telemetry snapshot,
   - topology relations,
   - reasoning,
   - recommended action.

Why modal:

- Diagnostic content is too dense for a sidebar.
- Incident response needs focused full-screen reading.

## `frontend/src/components/CameraIndicator.jsx`

Purpose: render a camera marker in 3D.

Workflow:

- Use camera position/lookAt.
- Render camera body/cone/label.
- Orient marker toward target.

Why visual cameras:

- Helps explain why some people are visible to certain feeds.

## `frontend/src/components/SensorIndicator.jsx`

Purpose: render sensor markers in 3D.

Workflow:

- Pick color by sensor type.
- Render marker and hover label.

Why sensor markers:

- Operators can connect telemetry anomalies to physical locations.

## `frontend/src/utils/zonePositions.js`

Purpose: frontend zone/equipment/camera/sensor geometry mirror.

Exports:

- `ZONES`: zone layout definitions for the 3D scene.
- `getZoneColor(status)`: maps status to color.
- `getZoneOpacity(status)`: maps status to overlay opacity.

Why this exists:

- The frontend needs fast, direct JS objects.

Tradeoff:

- This duplicates `cad/zone_definitions.json`.
- A future improvement would generate this file from JSON or load JSON at runtime.

