---
icon: lucide/settings
---

# Configuration Reference

The entire pipeline is driven by a single YAML file (default path `config/pipeline_config.yaml`). This page documents every section and key.

The config is parsed and validated with [Pydantic](https://docs.pydantic.dev/). **A key the configuration does not define is an error** at every level; the message names the key and its location (for an unknown top-level key it also lists the known top-level keys). The names under `systems`, `sampling_frequencies`, `system_descriptions` and `preprocessing.alternate_time_columns` must be tracking-system names, and every key under `validation.settings` must be declared by a registered check (see [`validation.settings`](#validationsettings)). Loading is done by `PipelineConfig.from_yaml(path)`, which raises a `ConfigurationError` if the file is missing, unparseable, empty, or fails validation. A run and `--dry-run` then apply the further checks listed under [Configuration errors](#configuration-errors), before any session is read.

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
| `data_dir` | path | `DATA` | Root directory containing session folders. It must be an existing folder; a relative path is read from the current folder. |
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
| `task_name` | string | `"VRtracking"` | `task-` entity used in filenames. Letters and digits only. |
| `overwrite` | bool (strict) | `false` | What a rerun into an existing output may overwrite (below). |

`overwrite: false` keeps a session's existing, non-empty `sourcedata/` copy (it is not copied again, and a warning says so) and the existing `README` and `dataset_description.json` files of the raw and derivative tiers. `overwrite: true` writes those files again and copies the input folder over the existing `sourcedata/` copy: files at the same path are overwritten, and files that are no longer in the input stay. Either way, the session files (motion, channels, events, scans), `participants.tsv`/`.json` and the reports are always rewritten, and `.bidsignore` is written only when absent.

---

## `session_mappings`

A list mapping each source folder to BIDS identifiers. If **omitted or empty**, the pipeline auto-discovers every direct subfolder of `data_dir` that contains a file matching `input.metadata_pattern`, in name order, and numbers subjects sequentially (`01`, `02`, …) with session `01`. If it finds none, the run ends with an error that names any deeper folders holding a metadata file (list those under `session_mappings`), and the exit code is `1`.

| Key | Type | Required | Description |
| --- | ---- | -------- | ----------- |
| `source_dir` | string | yes | Folder path **relative to `input.data_dir`**. |
| `subject_id` | string | yes | BIDS subject id → `sub-<id>`. Letters and digits only. |
| `session_label` | string | yes | BIDS session label → `ses-<label>`. Letters and digits only. |
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

The `sub-` and `ses-` prefixes are added for you, so write `subject_id: "01"`, not `"sub-01"`. Each `subject_id`/`session_label` pair and each folder may appear only once. `participants.tsv` has one row per subject, so every mapping of one subject must give the same `age`, `sex` and `handedness`, or none may give them; a value given in one mapping and left out of another counts as different, while an explicit `"n/a"` counts as left out. Each of these is a [configuration error](#configuration-errors).

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

`enabled` is a strict boolean and is required for each listed system. Names other than the six tracking systems are a configuration error.

---

## `sampling_frequencies`

Expected sampling rate (Hz) per system. **Required for every system that `systems` does not disable** (a system left out of `systems` counts as enabled), and every rate must be positive; otherwise the run stops with a configuration error before any session is read. An integer (`90`) is accepted and read as `90.0`.

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

Human-readable description per system, written to each `motion.json` as `TaskDescription` (unless overridden by `device.task_description`). Keys must be tracking-system names; a system without an entry gets `<System> tracking from <device.model_name>`.

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

BIDS dataset metadata. All keys required except `authors`, `readme_text` and `tracking_space_reference_frame`.

| Key | Type | Default (demo) | Description |
| --- | ---- | -------------- | ----------- |
| `missing_values` | string | `"n/a"` | Token written into TSV cells for missing data. |
| `dataset_type` | string | `"raw"` | `DatasetType` in the raw `dataset_description.json`. |
| `license` | string | `"CC0"` | Dataset license (data license, distinct from the Apache-2.0 code license). |
| `authors` | `list[string]` | `["ResXR Contributors"]` | `Authors` in `dataset_description.json` (omitted if empty). |
| `readme_text` | string | *(a short text about ResXR)* | Free-text body of the dataset `README`. If empty, the `README` holds the dataset name and "Generated by ResXR pipeline." |
| `reference_frame` | object | *(see below)* | Coordinate-system description for `channels.json`. |
| `tracking_space_reference_frame` | object | *(see below)* | The `tracking_space` level of `channels.json`, used only for hand-joint channels detected in that frame (see [BIDS Output](bids-output.md#channelstsv)). |

### `bids.reference_frame`

Describes the global coordinate system; emitted into every `channels.json` sidecar. All four keys required.

```yaml
reference_frame:
  description: "Global VR playspace coordinate system. Positive X is right, Y is up (superior), Z is forward (anterior)"
  rotation_rule: "left-hand"
  rotation_order: "ZXY"
  spatial_axes: "RSA"
```

### `bids.tracking_space_reference_frame`

Optional. Written to `channels.json` as the `tracking_space` level only when a session's hand-joint channels are detected in that frame. The defaults (also in the shipped config):

```yaml
tracking_space_reference_frame:
  description: "Meta tracking space as returned by the SDK: OpenXR right-handed axes, +X right, +Y up, -Z forward."
  rotation_rule: "right-hand"
  spatial_axes: "RSP"
  # rotation_order: when omitted, reference_frame.rotation_order is used
```

---

## `validation`

Configures which quality checks run and their thresholds. See [Validation & Quality Checks](validation.md) for the full behavior of each check.

| Key | Type | Default (demo) | Description |
| --- | ---- | -------------- | ----------- |
| `enabled_checks` | `list[string]` | `[hands_tracking_loss, sampling_rate, stats_summary, eyes_closed, clock_dropout]` | Names of checks to run, in order. An unknown name is a configuration error that lists the valid names. |
| `column_groups` | `list[object]` | `[]` | Named sets of columns for custom checks that read them; no built-in check does. |
| `settings` | object | `{}` | Thresholds and per-check group assignments, read by individual checks. Every key must be declared by a registered check. |

!!! warning "Thresholds live under `settings`"
    Tunable thresholds and check→group assignments must go **inside `settings:`**. A key placed directly under `validation:` is a configuration error.

### `validation.column_groups`

Each group has a `name` (required), `columns` (required, list of column names), and optional `description`. Group names must be unique. No built-in check reads column groups. A [custom check](validation.md#column-groups) gets its groups with `config.get_column_groups(check_name, default_columns=...)`: the groups that `settings.check_column_groups` assigns to it, or every group when it has no entry there; with no groups configured, a single default group holding the `default_columns` it passes.

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

The keys the built-in checks read (defaults apply when the key is absent). Each check declares the keys it reads in its `settings_keys` attribute, and `check_column_groups` is always accepted; any other key here is a configuration error that names the known keys. Keys inside a setting's value, such as the hand names under `tracking_flags`, are not checked.

| Key | Default | Used by | Meaning |
| --- | ------- | ------- | ------- |
| `sampling_rate_tolerance` | `0.10` | `sampling_rate` | Max fractional deviation of effective from expected rate before flagging (10%). |
| `sampling_cv_threshold` | `0.50` | `sampling_rate` | Max coefficient of variation of the intervals between the stream's own samples (50%). |
| `eyes_closed_threshold` | `0.9` | `eyes_closed` | Blendshape value (0–1) at/above which an eye counts as closed. |
| `eyes_closed_min_duration` | `0.1` | `eyes_closed` | Minimum closed duration (s) to flag. |
| `eyes_closed_use_min_duration` | `true` | `eyes_closed` | Whether to apply the minimum-duration filter. |
| `tracking_flags` | left/right `*_Status_HandTracked` | `hands_tracking_loss` | Map of hand → validity columns indicating tracking loss. |
| `check_column_groups` | — | custom checks that read column groups | Map of check name → list of group names to apply. Each group name must be defined in `column_groups`. |

```yaml
settings:
  sampling_rate_tolerance: 0.10
  sampling_cv_threshold: 0.5
  eyes_closed_threshold: 0.9
  eyes_closed_min_duration: 0.1
  check_column_groups:
    my_check:            # a custom check (see Validation → Writing a custom check)
      - "Left Hand"
      - "Right Hand"
```

---

## `preprocessing`

Controls optional quality masking and per-system time columns.

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `apply_quality_masking` | bool (strict) | `false` | When `true`, flagged segments are replaced with `NaN` in the **derivative** tier (the raw tier is never modified, and no rows are deleted). |
| `masking_checks` | `list[string]` \| `null` | `null` | `null` = apply all maskable flags; or list specific check names to restrict masking. With masking on, each name must also be in `validation.enabled_checks` (a configuration error otherwise). |
| `alternate_time_columns` | `map[string, string]` | `{}` | Per-system override of the time axis. Map a system (a tracking-system name) to exactly one column name. |

### `alternate_time_columns`

By default every stream uses the global engine clock as its `timestamp`. Some streams carry their own clock (e.g. hands sampled on a separate thread). Mapping a system to its own time column makes the splitter use that column as the stream's `timestamp` and preserve the global clock as `timeSinceStartup`. At export these become `latency` (per-system) and `latency_global` respectively — see [LATENCY channels](pipeline-stages.md#latency-channels).

Each configured column that a stream uses is checked for milliseconds and backward steps, with a warning naming the column; its values are not changed (see [Checks on the clocks](input-data.md#checks-on-the-clocks)). A configured column that the data do not have is a warning, and that stream is timed by `timeSinceStartup` alone.

```yaml
alternate_time_columns:
  Head: "Node_Head_Time"
  Hands: "Node_HandLeft_Time"
  Eyes: "Eyes_Time"
  Face: "Face_Time"
  Body: "Body_Time"
  Controllers: "Node_ControllerLeft_Time"
```

Leave it as `{}` only for data without per-system clock columns. Every stream is then timed by `timeSinceStartup` alone, and the `sampling_rate`, `clock_dropout`, `hands_tracking_loss` and `eyes_closed` checks, which place their flags on a separate `timeSinceStartup` column, log an error for each stream they check and raise no flags (they are listed as [not run](validation.md#checks-not-run)).

---

## `report`

| Key | Type | Default (demo) | Description |
| --- | ---- | -------------- | ----------- |
| `enabled` | bool (strict) | `true` | Generate the HTML quality report. Required — the `true` shown is the shipped demo value, not a code default. |
| `output_dir` | path \| null | `null` | Where to write the report. `null` = alongside the session's BIDS output. |

The report is named `<session_id>_report.html`. In a shared `output_dir`, two sessions with the same `session_id` therefore write one file (the second replaces the first), and a warning names both sessions.

See [Quality Reports](reports.md).

---

## Configuration errors

A run checks the configuration before any session is read, and `--dry-run` runs the same checks. Each of these problems stops the run with `Configuration error in <config file>: ...` and exit code `1`; nothing is written:

- the file cannot be read or parsed, or is empty; a required key is missing or a value has the wrong type (including a strict boolean that is not `true`/`false`);
- a key the configuration does not define, at any level;
- an unknown check name in `validation.enabled_checks` or `preprocessing.masking_checks`;
- with `apply_quality_masking: true`, a `masking_checks` name that is not in `validation.enabled_checks` (it never runs, so nothing would be masked for it);
- a key under `validation.settings` that no registered check declares;
- duplicate names in `validation.column_groups`; a `validation.settings.check_column_groups` value that is not a map of check names to lists of group names, or that names a group `validation.column_groups` does not define;
- an unknown tracking-system name under `systems`, `sampling_frequencies`, `system_descriptions` or `preprocessing.alternate_time_columns`;
- a sampling rate that is not positive, or a tracking system that `systems` does not disable without a `sampling_frequencies` entry;
- a `subject_id`, `session_label` or `output.task_name` that holds anything but letters and digits;
- two `session_mappings` with the same `subject_id` and `session_label`, or one folder mapped twice;
- `session_mappings` of one subject that give different `age`, `sex` or `handedness` values;
- an `input.data_dir` that is not an existing folder.

A config file that does not exist is reported as `Configuration file not found: <path>`, also with exit code `1`. Keys inside a setting's value (such as the hand names under `tracking_flags`, or the check names under `check_column_groups`) are not checked.
