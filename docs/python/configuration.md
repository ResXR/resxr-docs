---
icon: lucide/settings
---

# Configuration Reference

The entire pipeline is driven by a single YAML file (default path `config/pipeline_config.yaml`). This page documents every section and key.

The config is parsed and validated with [Pydantic](https://docs.pydantic.dev/). **Unknown keys are silently ignored** at every level (a warning lists unknown *top-level* keys), so a typo in a key name means your value is dropped rather than applied — watch the logs. Loading is done by `PipelineConfig.from_yaml(path)`, which raises a `ConfigurationError` if the file is missing, unparseable, empty, or fails validation.

!!! tip "Booleans are strict"
    The fields `output.overwrite`, `systems.<name>.enabled`, `preprocessing.apply_quality_masking`, and `report.enabled` are **strict booleans** — they must be a real YAML `true`/`false`, not `"true"`, `1`, or `yes`.

## Top-level structure

```yaml
bids: { ... }                  # BIDS metadata defaults
sampling_frequencies: { ... }  # expected Hz per system
system_descriptions: { ... }   # sidecar TaskDescription per system
input: { ... }                 # where to read from
output: { ... }                # where to write to
session_mappings: [ ... ]      # source folder → subject/session
device: { ... }                # hardware metadata
systems: { ... }               # enable/disable each system
validation: { ... }            # quality checks
preprocessing: { ... }         # masking & time columns
report: { ... }                # HTML report
```

`bids`, `input`, `output`, `device`, `validation`, `preprocessing`, `report`, `sampling_frequencies`, and `system_descriptions` are **required** sections. `session_mappings` and `systems` may be omitted (they default to empty).

---

## `input`

Where to read session data and how to find each file. All five keys are required.

| Key | Type | Default (demo) | Description |
| --- | ---- | -------------- | ----------- |
| `data_dir` | path | `DATA` | Root directory containing session folders. |
| `continuous_data_pattern` | string | `"*_ContinuousData*.csv"` | Glob for the main tracking CSV. |
| `face_data_pattern` | string | `"*_FaceExpressionData*.csv"` | Glob for the face tracking CSV. |
| `metadata_pattern` | string | `"*SessionMetadata.json"` | Glob for the session metadata JSON. |
| `events_data_pattern` | string | `"*_Events.csv"` | Glob for the optional events CSV. |

Patterns are resolved per session folder; if several files match, the most recent (by modification time) is used. See [Input Data Format](input-data.md).

---

## `output`

Where the BIDS dataset is written. All keys required.

| Key | Type | Default (demo) | Description |
| --- | ---- | -------------- | ----------- |
| `bids_root` | path | `output` | Output directory for the BIDS dataset. |
| `dataset_name` | string | `"ResXR VR Tracking Dataset"` | `Name` in `dataset_description.json`. |
| `bids_version` | string | `"1.10.1"` | BIDS spec version recorded in the dataset. |
| `task_name` | string | `"VRtracking"` | `task-` entity used in filenames. |
| `overwrite` | bool (strict) | `false` | Overwrite existing files when re-running. |

---

## `session_mappings`

A list mapping each source folder to BIDS identifiers. If **omitted or empty**, the pipeline auto-discovers every subfolder of `data_dir` containing a metadata file and numbers subjects sequentially (`01`, `02`, …) with session `01`.

| Key | Type | Required | Description |
| --- | ---- | -------- | ----------- |
| `source_dir` | string | yes | Folder path **relative to `input.data_dir`**. |
| `subject_id` | string | yes | BIDS subject id → `sub-<id>`. |
| `session_label` | string | yes | BIDS session label → `ses-<label>`. |
| `age` | string | no | Participant age → `participants.tsv` (`n/a` if unset). |
| `sex` | string | no | Participant sex (`M`/`F`) → `participants.tsv`. |
| `handedness` | string | no | `R`/`L`/`A` → `participants.tsv`. |

```yaml
session_mappings:
  - source_dir: "Demo_Data/Binary/2026.06.10_18-22"
    subject_id: "01"
    session_label: "01"
    age: "27"
    sex: "F"
    handedness: "R"
```

---

## `device`

Hardware metadata for the BIDS sidecars.

| Key | Type | Default (demo) | Description |
| --- | ---- | -------------- | ----------- |
| `manufacturer` | string | `"Meta"` | `Manufacturer` in `motion.json`. |
| `model_name` | string | `"Quest Pro"` | `ManufacturersModelName` in `motion.json`. |
| `task_description` | string | *(unset)* | Optional. If set, overrides the per-system `system_descriptions` for **all** motion files. |

---

## `systems`

Enable or disable each tracking system. A system absent from this map defaults to **enabled**. A system is only emitted if it is enabled here, enabled in the recording's metadata flags, *and* has data columns present.

```yaml
systems:
  Head:        { enabled: true }
  Hands:       { enabled: true }
  Eyes:        { enabled: true }
  Face:        { enabled: true }
  Body:        { enabled: false }
  Controllers: { enabled: false }
```

`enabled` is a strict boolean and is required for each listed system.

---

## `sampling_frequencies`

Expected sampling rate (Hz) per system. **Required for every system you emit** — splitting a system with no entry raises a `ConfigurationError`. Values must be floats (`90.0`, not `90`).

```yaml
sampling_frequencies:
  Head: 72.0
  Hands: 90.0
  Eyes: 30.0
  Face: 30.0
  Body: 72.0
  Controllers: 90.0
```

The expected rate is written as `SamplingFrequency`; the pipeline also measures the *effective* rate from the data and writes it as `SamplingFrequencyEffective`. The [`sampling_rate` check](validation.md#sampling_rate) compares the two.

---

## `system_descriptions`

Human-readable description per system, written to each `motion.json` as `TaskDescription` (unless overridden by `device.task_description`).

```yaml
system_descriptions:
  Head: "Head position and rotation tracking from Meta Quest VR headset"
  Hands: "Hand and finger tracking from Meta Quest VR headset including 26 joints per hand"
  Eyes: "Eye gaze tracking from Meta Quest VR headset"
  Face: "Facial expression tracking using FACS blend shapes from Meta Quest VR headset"
  Body: "Full body joint tracking from Meta Quest VR headset"
  Controllers: "VR controller position and rotation tracking"
```

---

## `bids`

BIDS dataset metadata. All keys required except `authors` and `readme_text`.

| Key | Type | Default (demo) | Description |
| --- | ---- | -------------- | ----------- |
| `missing_values` | string | `"n/a"` | Token written into TSV cells for missing data. |
| `dataset_type` | string | `"raw"` | `DatasetType` in the raw `dataset_description.json`. |
| `license` | string | `"CC0"` | Dataset license (data license, distinct from the Apache-2.0 code license). |
| `authors` | `list[string]` | `["ResXR Contributors"]` | `Authors` in `dataset_description.json` (omitted if empty). |
| `readme_text` | string | *(see below)* | Free-text body of the dataset `README`. |
| `reference_frame` | object | *(see below)* | Coordinate-system description for `channels.json`. |

### `bids.reference_frame`

Describes the global coordinate system; emitted into every `channels.json` sidecar. All four keys required.

```yaml
reference_frame:
  description: "Global VR playspace coordinate system. Positive X is right, Y is up (superior), Z is forward (anterior)"
  rotation_rule: "left-hand"
  rotation_order: "ZXY"
  spatial_axes: "RSA"
```

---

## `validation`

Configures which quality checks run and their thresholds. See [Validation & Quality Checks](validation.md) for the full behavior of each check.

| Key | Type | Default (demo) | Description |
| --- | ---- | -------------- | ----------- |
| `enabled_checks` | `list[string]` | `[hands_tracking_loss, sampling_rate, stats_summary, eyes_closed, clock_dropout]` | Names of checks to run, in order. Unknown names are skipped. |
| `column_groups` | `list[object]` | `[]` | Named sets of columns for column-scoped checks. |
| `settings` | object | `{}` | Free-form thresholds and per-check group assignments, read by individual checks. |

!!! warning "Thresholds live under `settings`"
    Tunable thresholds and check→group assignments must go **inside `settings:`**. Keys placed directly under `validation:` are silently ignored.

### `validation.column_groups`

Each group has a `name` (required), `columns` (required, list of column names), and optional `description`. Group names must be unique. Checks request their groups by name; if no groups are configured, a check falls back to a single default group containing all of the stream's columns.

```yaml
column_groups:
  - name: "Left Hand"
    description: "Left hand wrist position and rotation"
    columns:
      - Left_XRHand_Wrist_x
      - Left_XRHand_Wrist_y
      - Left_XRHand_Wrist_z
      - Left_XRHand_Wrist_Rotation_x
      - Left_XRHand_Wrist_Rotation_y
      - Left_XRHand_Wrist_Rotation_z
  - name: "Right Hand"
    description: "Right hand wrist position and rotation"
    columns:
      - Right_XRHand_Wrist_x
      # ...
```

### `validation.settings`

Common keys (each consumed by a specific check; defaults apply when the key is absent):

| Key | Default | Used by | Meaning |
| --- | ------- | ------- | ------- |
| `sampling_rate_tolerance` | `0.10` | `sampling_rate` | Max fractional deviation of effective from expected rate before flagging (10%). |
| `sampling_cv_threshold` | `0.50` | `sampling_rate` | Max coefficient of variation of inter-sample intervals (50%). |
| `eyes_closed_threshold` | `0.9` | `eyes_closed` | Blendshape value (0–1) at/above which an eye counts as closed. |
| `eyes_closed_min_duration` | `0.1` | `eyes_closed` | Minimum closed duration (s) to flag. |
| `eyes_closed_use_min_duration` | `true` | `eyes_closed` | Whether to apply the minimum-duration filter. |
| `tracking_flags` | left/right `*_Status_HandTracked` | `hands_tracking_loss` | Map of hand → validity columns indicating tracking loss. |
| `check_column_groups` | — | any column-scoped check | Map of check name → list of group names to apply. |

```yaml
settings:
  sampling_rate_tolerance: 0.10
  sampling_cv_threshold: 0.5
  eyes_closed_threshold: 0.9
  eyes_closed_min_duration: 0.1
  check_column_groups:
    hands_tracking_loss:
      - "Left Hand"
      - "Right Hand"
```

---

## `preprocessing`

Controls optional quality masking and per-system time columns.

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `apply_quality_masking` | bool (strict) | `false` | When `true`, flagged segments are replaced with `NaN` in the **derivative** tier (the raw tier is never modified, and no rows are deleted). |
| `masking_checks` | `list[string]` \| `null` | `null` | `null` = apply all maskable flags; or list specific check names to restrict masking. |
| `alternate_time_columns` | `map[string, string]` | `{}` | Per-system override of the time axis. Map a system to exactly one column name. |

### `alternate_time_columns`

By default every stream uses the global engine clock as its `timestamp`. Some streams carry their own clock (e.g. hands sampled on a separate thread). Mapping a system to its own time column makes the splitter use that column as the stream's `timestamp` and preserve the global clock as `timeSinceStartup`. At export these become `latency` (per-system) and `latency_global` respectively — see [LATENCY channels](pipeline-stages.md#latency-channels).

```yaml
alternate_time_columns:
  Head: "Node_Head_Time"
  Hands: "Node_HandLeft_Time"
  Eyes: "Eyes_Time"
  Face: "Face_Time"
  Body: "Body_Time"
  Controllers: "Node_ControllerLeft_Time"
```

Leave it as `{}` to use the global clock for all systems.

---

## `report`

| Key | Type | Default (demo) | Description |
| --- | ---- | -------------- | ----------- |
| `enabled` | bool (strict) | `true` | Generate the HTML quality report. Required — the `true` shown is the shipped demo value, not a code default. |
| `output_dir` | path \| null | `null` | Where to write the report. `null` = alongside the session's BIDS output. |

See [Quality Reports](reports.md).
