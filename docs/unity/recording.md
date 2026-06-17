---
icon: lucide/circle-dot
title: Recording
description: What is recorded, how to enable each subsystem, reference frames, and sampling.
---

# Recording

All recording is driven by the **`ResXRDataManager`** component, which lives on the `ResXR_DataManager` prefab in the Base Scene. It owns a set of **collectors** — one per tracking subsystem — and samples the enabled ones once per physics tick, writing a row to `*_ContinuousData.csv`. Face data is written to its own file. This page covers what each subsystem records, how to turn it on or off, the coordinate frames the numbers are in, and the sampling cadence.

## Subsystems and their toggles

Most toggles live in the **What to record (ContinuousData)** group on `ResXRDataManager`, inside a `RecordingOptions` block. Face is a separate top-level checkbox. Turning a subsystem off removes its collector and its columns entirely (and records that fact in the [metadata](data-output.md#session-metadata)).

| Subsystem | Inspector toggle | Collector | What it records |
| --------- | ---------------- | --------- | --------------- |
| **Nodes** (head, controllers, hand/eye-center positions) | `Include Nodes` | `OVRNodesCollector` | Per node (`Head`, `HandLeft`, `HandRight`, `ControllerLeft`, `ControllerRight`, `EyeCenter`): position, rotation, and presence/valid/tracked flags. |
| **Eyes — orientation** | `Include Eyes` | `OVREyesCollector` | Per-eye gaze orientation quaternions (rotation values), validity, and confidence. |
| **Eyes — combined gaze** | `Include Gaze` | `OVREyesCollector` | The cyclopean (averaged) gaze ray: the world-space hit point and the focused object's name. |
| **Eyes — per-eye gaze** | `Include Separate Eyes Gaze` | `OVREyesCollector` | Separate left/right gaze hit points and focused objects. Also makes the eye tracker cast three rays per frame instead of one. |
| **Hands** | `Include Hands` | `OVRHandsCollector` | Per hand: tracking status flags, root pose, scale, overall and per-finger confidence, and every hand-skeleton bone pose. |
| **Body** | `Include Body` | `OVRBodyCollector` | Full-body skeleton: per-joint pose and tracking flags, plus confidence, fidelity, and calibration status. |
| **System status** | `Include System Status` | `SystemStatusCollector` | Recenter events and counts, tracking-origin changes, the tracking transform, user-present, and tracking-lost. |
| **Performance** | `Include Performance` | `OVRPerformanceCollector` | Reserved — currently writes **no columns** (the motion-to-photon latency column was removed from the schema; the collector is kept for future metrics). Returns empty on the OpenXR backend. |
| **Custom transforms** | `Custom Transforms To Record` (object list) | `CustomTransformsCollector` | World position and rotation of any scene objects you drag in — one set of columns per object. |
| **Face** | `Record Face Expressions` | `OVRFaceCollector` | Facial-expression activations and per-region confidence, written to `*_FaceExpressionData.csv`. |

!!! info "Blendshape / facial-expression weights"
    The face collector records ~70 **expression weights** — numeric activations (0–1) of individual facial movements such as a brow raise or a jaw drop, following Meta's Face Tracking set. (A *blendshape* is the mesh deformation each weight would drive on an avatar.) They are written to the dedicated face file, not the continuous file.

!!! warning "Body tracking is an estimate, not a body sensor"
    Quest headsets have no body sensor. With `Include Body` on, Meta's SDK *infers* a full-body skeleton from head and hand poses for smooth avatar animation — it is not ground-truth motion capture. The template ships with it on, but the in-editor tooltip recommends leaving it **off** unless your analysis genuinely needs the estimated joints. Treat body joints accordingly.

### Enabling a subsystem

1. Select the `ResXR_DataManager` object in the Base Scene.
2. In the Inspector, tick the toggle for the subsystem (or add objects to **Custom Transforms To Record**).
3. Make sure the hardware supports it — eye, face, and body tracking require the relevant Quest features and permissions; see [Installation](installation.md#quest-device-setup).

Controllers do not have a dedicated toggle: their poses come in through **Nodes** (`ControllerLeft` / `ControllerRight`). The metadata always reports `controllers_enabled` as true.

## Reference frames

Tracking data from the Meta runtime arrives in **tracking space** — a right-handed coordinate system whose origin is the play-area boundary you set up on the headset. Unity's **world space** is left-handed. If the two were mixed, positions and rotations would not line up.

`ResXRDataManager` resolves this with the static `TrackingSpaceConverter` utility, initialized at startup from the `OVRCameraRig` (Meta's headset-and-controllers rig object in the scene). Every world-space position and rotation written to `*_ContinuousData.csv` — except hand-bone poses, which stay hand-local (see below) — is passed through it, so those columns are consistent Unity world-space values regardless of recenter events during the session. Two frames are used:

- **UnityWorld** — left-handed, Euler order `ZXY`, axes `+X right, +Y up, +Z forward`. This is the frame for node, gaze-hit, body, and custom-transform columns.
- **HandLocal** — right-handed, Euler order `ZXY`, hand-local axes relative to the hand root. Hand **bone** poses stay in this frame (they are not converted to world space) because they are most useful relative to the wrist.

Both frames, the Euler order (`ZXY`), the rotation units (`degrees`), and the headset's `tracking_origin_type` (`EyeLevel`, `FloorLevel`, or `Stage`) are written into `*_SessionMetadata.json` so the pipeline can describe each channel correctly. The conversion math is documented in `Assets/ResXR/Base Scene/ResXRDataManager/Utilities/TRACKING_TO_WORLD_CONVERSION.md`.

!!! note "Quaternions and Euler angles"
    Rotations are stored as **quaternions** in the continuous data (the `_qx/_qy/_qz/_qw` columns) because they are unambiguous and interpolate cleanly. The `ZXY` Euler order in the metadata tells downstream tools how to decompose them into angles if needed.

## Sampling

Continuous data is sampled in Unity's `FixedUpdate`, the fixed-rate physics step. The default **Fixed Timestep is `0.01` s — 100 Hz**. Each tick, every enabled collector fills one row buffer and the row is written to disk and flushed immediately, so a crash or unclean shutdown loses at most the current tick.

The timeline column is **`timeSinceStartup`** (seconds since the app started, from `Time.realtimeSinceStartup`, unaffected by pauses or time scaling). It is shared by the continuous and face files, and it is the clock your event logging should use as well.

To change the rate, edit **Project Settings → Time → Fixed Timestep** (this is a Unity-wide setting, not a ResXR field). The chosen `fixedDeltaTime` and `sampling_mode` are recorded in the metadata; the pipeline turns `timeSinceStartup` into BIDS [latency channels](../python/pipeline-stages.md#latency-channels) at export.

!!! warning "Higher rates cost frame time"
    Every enabled subsystem adds work to each tick. Recording hands, body, and per-eye gaze at a high Fixed Timestep can affect frame rate on the headset. Enable only the subsystems your study needs, and see [Extending → Performance notes](extending.md#performance-notes).

For the exact files and columns these settings produce, see [Data Output](data-output.md).
