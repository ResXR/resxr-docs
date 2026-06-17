---
icon: lucide/file-input
---

# Input Data Format

This page describes the **data contract**: the files the Unity/Meta Quest recorder writes, and how the pipeline reads them. If you are recording with a different engine, matching this layout is all that is required — there is no shared code between the recorder and the pipeline.

!!! info "Producing these files from Unity"
    This page is the authoritative specification of the format. For how the bundled Unity template *writes* these files — naming, on-device location, the recording subsystems, and how to add your own tables — see [Unity Template → Data Output](../unity/data-output.md).

## Session folder layout

The pipeline processes one **session folder** at a time. A session folder contains the files for a single recording, conventionally prefixed with the session id:

```
session_folder/
├── *_ContinuousData.csv       # Main per-frame tracking (head, hands, eyes, …)   [REQUIRED]
├── *_FaceExpressionData.csv   # Face tracking (FACS blendshapes)                  [optional]
├── *_Events.csv               # Task / stimulus event markers                     [optional]
├── *_SessionMetadata.json     # Recording configuration & timestamps             [REQUIRED]
└── *_CustomTables/            # Experiment-defined custom data tables             [optional]
    ├── *_CustomTables.json     # Schema describing each table's columns
    └── *_<ClassName>.csv       # One CSV per custom data class
```

Files are located by **glob pattern** — a filename wildcard such as `*_ContinuousData*.csv`, where `*` matches any characters — configured under `input:` in the [config](configuration.md#input). If a pattern matches multiple files in a folder, the pipeline uses the **most recently modified** one and logs a warning.

!!! warning "Required vs optional"
    `*_SessionMetadata.json` and `*_ContinuousData*.csv` are **required** — a session is skipped with an error if either is missing. Face, events, and custom tables are optional. A malformed *face* file is downgraded to a warning (face data dropped); a malformed *events* or *continuous* file raises an error.

## Continuous data

`*_ContinuousData.csv` is the main per-frame recording — one row per engine frame, often hundreds of columns wide (the demo sessions have ~530 columns). It is read with UTF-8 BOM handling and the missing-value tokens `""`, `NaN`, `null`, `None`.

The recorder's global clock column, **`timeSinceStartup`**, is renamed on load to **`timestamp`** and becomes the canonical time axis. A continuous file with no `timeSinceStartup`/`timestamp` column is rejected.

### How columns map to tracking systems

The pipeline classifies every column to a tracking system by matching **column-name prefixes**. You don't list columns anywhere — adding a new column with a recognized prefix automatically routes it to the right system.

| System | Column prefixes (examples) |
| ------ | -------------------------- |
| **Head** | `Node_Head_*`, `FocusedObject`, `RecenterCount`, `TrackingLost`, `UserPresent`, `recenterEvent`, `shouldRecenter`, `timeSinceStartup`, `TrackingOriginChange_*`, `TrackingTransform_*` |
| **Hands** | `Node_HandLeft_*`, `Node_HandRight_*`, `LeftHand_*`, `RightHand_*`, `Left_XRHand_*`, `Right_XRHand_*` |
| **Eyes** | `EyeGazeHitPosition_*`, `LeftEye_*`, `RightEye_*`, `Node_EyeCenter_*`, `Eyes_Time`, `Left/RightEyeGazeHitPosition_*`, `Left/RightFocusedObject`, `HasLeft/RightEyeHit` |
| **Face** | `Face_*`, `Brow_*`, `Cheek_*`, `Chin_*`, `Dimpler`, `Eyes_Closed`, `Eyes_Look`, `Inner_Brow`, `Jaw_*`, `Lid_*`, `Lip_*`, `Lips_*`, `Lower_Lip`, `Mouth_*`, `Nose_*`, `Outer_Brow`, `Upper_Lid`, `Upper_Lip`, `Tongue_*`, `FaceRegionConfidence` (the complete list is `SYSTEM_COLUMN_PREFIXES[Face]` in `core/constants.py`) |
| **Body** | `Body_*` (e.g. `Body_Time`, `Body_Confidence`, `Body_Fidelity`, `Body_CalibrationStatus`) |
| **Controllers** | `Node_ControllerLeft_*`, `Node_ControllerRight_*` |

A system is only emitted if at least one of its columns is present (more than just the time column).

!!! note "Face is read from its own file"
    Although `Face_*` columns are recognized in the continuous data, the **Face** tracking system is built from the dedicated `*_FaceExpressionData.csv` file, not the continuous CSV.

## Face expression data

`*_FaceExpressionData.csv` holds FACS blendshape values — one numeric column per facial-expression activation (FACS = the Facial Action Coding System; the demo sessions carry ~70 expressions). It follows the same time-column convention (`timeSinceStartup` → `timestamp`). A `Face_Status` column, if present, is normalized from string/textual booleans (`"true"/"1"` → `True`, `"false"/"0"` → `False`); values that can't be mapped become `NaN` and are reported.

## Events

`*_Events.csv` carries task and stimulus markers. It must include these columns:

| Column | Type | Description | Example |
| ------ | ---- | ----------- | ------- |
| `onset` | float | Event start time in seconds (relative to session start). | `2.5`, `5.832` |
| `duration` | float | Event duration in seconds; `0` for an instantaneous event. | `1.5`, `0` |
| `name` | string | Event type/label; exported to BIDS as `trial_type`. | `stimulus_onset`, `response` |

```csv
onset,duration,name
0.0,0,trial_start
2.5,1.5,stimulus_A
4.2,0,response
5.0,0,trial_end
```

`onset` and `duration` are coerced to float (non-numeric values are an error), and rows are sorted by `onset`. At export, events are merged with any [custom tables](#custom-tables) into a single `events.tsv` at the session root. See [BIDS Output → Events](bids-output.md#events).

## Session metadata

`*_SessionMetadata.json` describes the recording. Every field is read defensively with sensible fallbacks; only `session_id` is essential. Captured fields include:

- **Identity & timing** — `session_id`, `utc_start_iso8601` (parsed to a UTC datetime; a trailing `Z` is handled), `device_utc_offset`, `sampling_mode`, `fixedDeltaTime` (the pipeline falls back to `0.02` s if the key is absent; the ResXR recorder writes the real value, typically `0.01` s / 100 Hz), `schema_rev`.
- **Feature flags** — `face_enabled`, `body_enabled`, `hands_enabled`, `eyes_enabled`, `controllers_enabled`. These gate which systems are split out.
- **Schema counts** — `schema_hand_bones`, `schema_body_joints`, `schema_face_expressions` (with legacy `detected_*` fallbacks).
- **Device** — `platform`, `build_id`, `device_serial_number`.

### Software and version metadata

Software/version capture is **engine-agnostic**. The loader collects *every* top-level metadata key whose name contains `"version"` (case-insensitive) and whose value is a non-empty string or number — for example `unity_version`, `ovrplugin_runtime_version`, `horizon_os_version`, `software_versions_raw`.

These captured versions are:

- listed in the HTML report header under **Software versions**, and
- folded into the BIDS `SoftwareVersions` sidecar field.

Known vendor keys (Unity, OVR Plugin, Horizon OS, the raw OS build string) get curated labels and formatting; any other recorder's keys flow through with an auto-generated label (e.g. an Unreal session's `unreal_engine_version` becomes "Unreal Engine …"). **Supporting a new headset therefore needs no code changes** — you only add a table entry if you want a prettier label or to filter sentinel values.

## Custom tables

Custom tables let an experiment record arbitrary structured data alongside the standard streams — trial parameters, stimulus bounds, ratings, reaction times, and so on. They live in a `<session_id>_CustomTables/` subfolder:

- **`*_CustomTables.json`** — a schema. A top-level `CustomTables` object maps each table name to its `RowCount` and a `Columns` object. Every column declares a `Description` and `Format` (required) plus optional `Units`, `Levels` (enum mapping), `Minimum`, and `Maximum`.
- **One CSV per table** — named `<session_id>_<ClassName>.csv`. Each must include `onset` and `duration` columns. These CSVs are read as **strings** to preserve exact source tokens (e.g. `false`, `-0.752`); only `onset`/`duration` are coerced to float for sorting.

The schema and CSVs are cross-checked: a declared table with no CSV, or a row-count mismatch, produces a warning. At export, custom-table rows are merged into the session `events.tsv` and their column descriptions flow into the `events.json` sidecar. See [BIDS Output → Events](bids-output.md#events).

!!! example "Custom table example (Binary Choice — `ChoiceEvents`)"
    ```csv
    onset,duration,Task,Trial,OptionAName,OptionBName,Choice,ChosenOption,HandUsed,ReactionTime,displayTime,ChoiceTime
    15.36217,0,1 Fixed Order Round Example,0,Snack_001,Snack_002,Snack_001,A,Left,1.846298,13.51224,15.35854
    ```

## Time columns: `timestamp` and `timeSinceStartup`

Two time concepts run through the pipeline:

- **`timestamp`** — the canonical per-stream time axis. By default this is the global engine clock (`timeSinceStartup` renamed). If you configure an [alternate per-system time column](configuration.md#preprocessing) (e.g. `Node_HandLeft_Time` for Hands), that column becomes the stream's `timestamp` instead.
- **`timeSinceStartup`** — the global engine clock, preserved alongside `timestamp` whenever a per-system time column is in use.

Neither column is written to BIDS verbatim. At export they become the BIDS [LATENCY channels](pipeline-stages.md#latency-channels) `latency` and `latency_global`. Validation flags and the report express all times relative to the **recording onset**, the first non-zero timestamp, so leading/trailing zero-padding rows never count.
