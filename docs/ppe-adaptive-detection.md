# Adaptive PPE Detection — Design Context & Handoff

**Status:** Design exploration (not yet implemented)
**Owner:** RigVision-3D team
**Purpose:** Self-contained brief so this idea can be continued in a fresh session without prior chat history.

---

## 1. Project context (one paragraph)

RigVision-3D is a real-time 3D digital-twin for monitoring ONGC drilling rigs. Data flows:
`cameras → CV pipeline (Python, YOLOv8 + BoT-SORT + RTMPose) → Redis → FastAPI/WebSocket → React/Three.js dashboard`.
The CV pipeline detects persons + PPE, tracks them, triangulates 3D positions, and writes person/zone state to Redis. PPE compliance status per person is part of the core `rigvision:persons` contract. Stack already includes **ChromaDB** (vector store, Pillar 5) and an LLM layer (Gemini/Claude) intended for RAG reasoning.

---

## 2. The problem

PPE categories must be **extensible at runtime**. Initially ~3 items (hard hat, vest, goggles), but more (gloves, ear protection, face shield, respirator, boots, harness, knee pads) may be added at any time. **We do NOT want to retrain/fine-tune the model every time an item is added.**

---

## 3. Current implementation & why it's rigid

PPE today uses a **closed-vocabulary** detector — class count is baked into YOLOv8's output head, so a new class = retrain.

Key code anchors (`cv/detection/detector.py`):
- `PersonDetector.cls_mapping = {"person":0, "hardhat":1, "vest":2, "goggles":3}` (~L156) — **hardcoded class list = the rigidity.**
- `is_ppe_model` auto-detected by class names (~L153).
- `_associate_ppe_to_persons(...)` (~L239) — assigns PPE boxes to persons by **bbox containment > 0.7**. This step is class-agnostic in spirit but has a crowd failure mode (can assign person A's hard-hat box to person B on overlap).
- `_classify_posture(bbox, keypoints)` (~L360) — already indexes **COCO-17 keypoints** (`kpts[5]`, `kpts[11]`, etc.).
- RTMPose-L (`rtmlib.Body`) initialized ~L168–186; pose inference at ~L335: `keypoints_all, scores_all = self.pose_model.pose_model(orig_img, bboxes=bboxes)`.
  - ⚠️ Note: `scores_all` (per-keypoint confidence) is currently **discarded** — only `det.keypoints` is stored. The adaptive design needs these scores → small change to retain them.
- `detect_and_recognize(...)` (~L196) — QR-based personnel ID (pattern to mirror for a prototype gallery).

Downstream coupling (do not break):
- `Detection` dataclass has `ppe: Optional[dict]` and `keypoints` fields (~L43).
- `tracker.py` carries `ppe` forward; **in non-PPE/COCO mode it FAKES PPE from `track_id % 4`** (~L266) — remember this when demoing.
- `pipeline.py` uses `track.ppe["hardhat"]` for box color and writes the `ppe` dict to Redis.
- **Redis contract** `rigvision:persons`: `"ppe": {"hardhat": bool, "vest": bool, "goggles": bool}`. Any redesign should keep emitting a `ppe` dict (ideally now dynamic keys + an `"unknown"` state).
- `infra/postgres/init.sql`: `person_tracking` has `ppe_hardhat/vest/goggles` columns; `violations` has an `evidence_frame BYTEA` column (good target for VLM-explained evidence).

---

## 4. Requirements / constraints

- Real-time: 3 cameras, ~10 Hz output, RTX 4070.
- Add a PPE item **without training** (config + a few example images at most).
- PPE is **safety-critical** → per-item threshold calibration + validation before an item gates a real decision; occlusion must not be reported as a violation.
- Keep the Redis `ppe` contract and existing tracking/triangulation untouched where possible.

---

## 5. Options considered (summary)

- **A. Open-vocabulary detector** — classes as text at inference.
  - **YOLO-World** ⭐ (Ultralytics family, real-time on 4070, drop-in-ish: `model.set_classes([...])`). Best speed/effort fit for *localization*.
  - Grounding DINO (accurate, too slow for live → good offline labeler), OWLv2 (text + image-example queries, slower).
- **B. Two-stage: person detect → pose-guided region crops → CLIP/SigLIP zero-shot.** Leverages existing RTMPose. Add item = add a text prompt + region + threshold.
- **C. Few-shot prototype gallery (enroll, don't train).** Embed a few example crops per item (CLIP/SigLIP/DINOv2), store prototypes, nearest-neighbor at inference. Add item = enroll photos. (This is the retrieval half of VRAG.)
- **D. Config-driven PPE registry (do regardless).** YAML/JSON lists items {id, method, prompt/anchor, region, threshold, required_in zones}. Pipeline reads it → "add item" becomes config edit + reload, not a code change.
- **E. Hybrid: open-vocab auto-labels → human review → periodic fine-tune of a fast YOLOv8.** Flexibility now, accuracy+speed later (pseudo-label/distill).

**VRAG (Visual RAG)** = retrieval over a visual knowledge base instead of retraining. Two flavors:
- **Flavor 1 — retrieval-as-classifier (fast):** embed crop → ChromaDB top-k → vote. Fits the 10 Hz path. (= Option C.)
- **Flavor 2 — retrieval-augmented VLM reasoning (rich, slow):** retrieve labeled exemplars → prompt a VLM to decide + explain. **Not in the hot loop** — use async/on-demand for violation verification, low-confidence cases, evidence captioning, and Pillar-5 reasoning.

---

## 6. Chosen direction — keypoint-anchored regions + embedding/VRAG retrieval

Core idea: **don't make the model *know* the classes — make it *look up* the right body region.** Each PPE device has an expected anatomical location ("where each device must sit"). RTMPose-L provides those anchors. Per person, per frame: keypoints → region boxes → embed crop → ChromaDB nearest-neighbor against a prototype gallery → label. Adding a device = define its anchor + drop in exemplars. **No PPE detector, no retraining.**

This **replaces** both the YOLO PPE classes and the `_associate_ppe_to_persons` containment matcher.

### Region map (COCO-17 keypoint indices)

COCO-17: `0 nose, 1 L-eye, 2 R-eye, 3 L-ear, 4 R-ear, 5 L-shoulder, 6 R-shoulder, 7 L-elbow, 8 R-elbow, 9 L-wrist, 10 R-wrist, 11 L-hip, 12 R-hip, 13 L-knee, 14 R-knee, 15 L-ankle, 16 R-ankle.`

| PPE device | Anchor keypoints | Region to crop |
|---|---|---|
| Hard hat | 0,1,2,3,4 | box above & around head, sized from ear-to-ear width |
| Goggles / mask / face shield | 0–4 | tight face box |
| Ear protection | 3, 4 | small box at each ear |
| Hi-vis vest / harness | 5,6 + 11,12 | torso quad |
| Gloves | 9, 10 | small box at each wrist |
| Safety boots | 15, 16 | box at each ankle |
| Knee pads | 13, 14 | box at each knee |

### Architecture

```
RTMPose-L keypoints (+per-kpt scores)        [already produced; scores currently discarded]
        │
   region boxes per configured device  ──────  scale-normalized, confidence-gated
        │
   embed crop (SigLIP / CLIP / DINOv2)
        │
   ChromaDB query → top-k exemplars + metadata [vector store already in stack]
        │
   ┌────┴───────────────────────────┐
   │                                │
Flavor 1: NN vote (per-frame,10Hz)  Flavor 2: VLM reason (async/on-demand)
   │                                │
   └─► ppe{} → Redis                └─► violation evidence + KG link (Pillar 5)
```

### Why it's better than current bbox-containment
- No mis-assignment in crowds (anchored to the specific skeleton).
- No PPE detector needed — localize from pose alone (the "no-retrain" property).
- "Missing PPE" falls out naturally: per region store `wearing` vs `bare` prototypes; nearest = the compliance boolean.
- Posture-aware for free (head box tracks the head even when bending/lying — posture already classified).

---

## 7. Engineering details that make/break it

1. **Retain & gate on per-keypoint confidence.** RTMPose returns `scores_all`; currently dropped. Keep them. If a region's anchor keypoints are low-confidence (occluded), emit `"unknown"` — **never** report occlusion as a violation (trust killer).
2. **Scale-relative crop sizing.** Size boxes from body scale (ear-to-ear distance / shoulder width), NOT fixed pixels — must survive people walking near/far. Biggest robustness lever given far cameras + 640×480.
3. **Two prototypes per region** (`wearing` / `bare`), classify by nearest — better calibrated than a single threshold.
4. **Multi-view fusion.** Cross-camera matching already exists; a glove occluded in cam0 may be visible in cam1 → take the higher-confidence per-region view.
5. **Config-driven registry (Option D).** Items defined in YAML: `{id, anchor_keypoints, region, embed_method, threshold, required_in_zones}`. Pipeline reads it; Redis `ppe` keys + compliance rules become dynamic.

---

## 8. Honest caveats
- **Small crops at distance** (hands/gloves, ankles) can be a handful of pixels → unreliable embeddings. Strongest for head/face/torso; weakest for extremities.
- **Self-occlusion** (hands behind back) → rely on confidence gating + multi-view; accept some `"unknown"`.
- **Embedding domain gap** — pre-trained CLIP on tiny/blurry crops is weak; SigLIP/DINOv2 usually better, and good in-situ exemplars from the actual cameras matter more than model choice.
- **Latency tiering is mandatory** — retrieval (Flavor 1) on the live path; VLM (Flavor 2) async only.
- **Safety-critical** — calibrate & validate each item before it gates a decision.

---

## 9. Assets already in the repo to reuse
- RTMPose-L keypoints: `cv/detection/detector.py` (init ~L168, inference ~L335, COCO indices used in `_classify_posture` ~L384).
- `Detection`/`TrackedPerson` already carry `keypoints` and `ppe`.
- Cross-camera matching: `cv/tracking/cross_camera.py` (for multi-view fusion).
- Vector store: ChromaDB (Pillar 5 in `CLAUDE.md`).
- Evidence storage: `violations.evidence_frame` in `infra/postgres/init.sql`.
- Personnel-ID gallery pattern: `detect_and_recognize` (QR) in `detector.py` — mirror for prototype enrollment.

---

## 10. Proposed next steps (prototype order)

1. **Retain keypoint scores** from RTMPose (stop discarding `scores_all`); add `keypoint_scores` to `Detection`.
2. **Region-extraction function** — `keypoints (+scores) → {device: crop_box}`, scale-normalized + confidence-gated. Wire behind the existing `Detection`/`_associate_ppe_to_persons` interface so the Redis contract is unchanged. *(This is the foundational piece; everything else builds on it.)*
3. **PPE registry config** (Option D) — YAML defining devices/anchors/regions/thresholds.
4. **Embedding + ChromaDB lookup** (VRAG Flavor 1) — SigLIP embed crop → nearest-neighbor vs `wearing`/`bare` prototypes → `ppe` dict (now dynamic keys + `"unknown"`).
5. **Enrollment tool** — add an item by dropping in a few example crops + re-embed (no training).
6. **(Later) VLM verification (Flavor 2)** — async, on flagged violations / low-confidence; writes explained evidence + KG link.
7. **(Optional) YOLO-World** for localization if pose-anchored crops prove insufficient for some items; or Option E distill path for safety-critical items needing top accuracy.

---

## 11. Open decisions
- Embedding model: SigLIP vs CLIP vs DINOv2 (lean SigLIP/DINOv2 for small crops).
- Keep YOLOv8 for person detection (yes) — only PPE localization moves to pose anchors.
- Per-device confidence + similarity thresholds (need a labeled validation set from the real cameras).
- How to represent dynamic PPE keys in Redis + Postgres without breaking the current fixed `hardhat/vest/goggles` columns (migration vs JSON column).
