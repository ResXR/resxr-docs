---
icon: lucide/folder-tree
---

# BIDS Output

The pipeline writes a [Motion-BIDS](https://bids.neuroimaging.io/) dataset. This page catalogs every file it produces and the conventions behind them.

!!! info "Two BIDS terms used throughout this page"
    - **Entity** — a `key-value` segment in a BIDS filename, e.g. `sub-01`, `ses-01`, `task-VRtracking`, `tracksys-Head`. Entities encode what a file is, so the name alone identifies its subject, session, task, and tracking system.
    - **Sidecar** — a `.json` file sitting next to a data file with the same name, holding that file's metadata (units, sampling rate, descriptions). Every `motion.tsv`/`channels.tsv` has a matching `.json` sidecar.

## Directory structure

A run produces three tiers under `bids_root`: the **raw** dataset at the root, a verbatim **`sourcedata/`** copy of the inputs, and a preprocessed **`derivatives/resxr/`** tier.

```
bids_root/
├── dataset_description.json
├── participants.tsv
├── participants.json
├── README
├── .bidsignore
├── sub-01/
│   └── ses-01/
│       ├── sub-01_ses-01_scans.tsv
│       ├── sub-01_ses-01_task-VRtracking_events.tsv
│       ├── sub-01_ses-01_task-VRtracking_events.json
│       ├── <session_id>_report.html
│       └── motion/
│           ├── sub-01_ses-01_task-VRtracking_tracksys-Head_motion.tsv
│           ├── sub-01_ses-01_task-VRtracking_tracksys-Head_motion.json
│           ├── sub-01_ses-01_task-VRtracking_tracksys-Head_channels.tsv
│           ├── sub-01_ses-01_task-VRtracking_tracksys-Head_channels.json
│           └── ... (Hands, Eyes, Face, …)
├── sourcedata/
│   └── sub-01/ses-01/        # exact copy of the input session folder
└── derivatives/
    └── resxr/
        ├── dataset_description.json     # DatasetType: derivative
        ├── README
        └── sub-01/ses-01/
            ├── sub-01_ses-01_scans.tsv
            └── motion/ ...              # same motion/channels files (no events)
```

One `tracksys-` set (`motion.tsv` + `motion.json` + `channels.tsv` + `channels.json`) is written per enabled tracking system. The derivative tier mirrors the motion files but omits the events files.

## Filenames

Motion and channels files follow the BIDS entity convention:

```
sub-<subject>_ses-<session>_task-<task>_tracksys-<System>_<suffix>.<ext>
```

For example, with `task_name: VRtracking`:

```
sub-01_ses-01_task-VRtracking_tracksys-Head_motion.tsv
sub-01_ses-01_task-VRtracking_tracksys-Hands_channels.tsv
```

`<System>` is one of `Head`, `Hands`, `Eyes`, `Face`, `Body`, `Controllers`. The `scans.tsv` and `events.tsv` files use only `sub-`/`ses-` (and `task-` for events), without `tracksys-`.

## `motion.tsv`

The numeric time series for one tracking system — **one row per sample, no header row** (the columns are described in the sibling `channels.tsv`). The column order matches `channels.tsv` exactly.

Each `motion.tsv` is produced from a [prepared DataFrame](pipeline-stages.md#latency-channels): its leading columns are `latency` (and `latency_global` when a per-system clock is used), and the internal `timestamp`/`timeSinceStartup` columns have been removed. Missing values are written using the configured token (default `n/a`); files use LF line endings and full float precision.

## `channels.tsv`

One row per column in the matching `motion.tsv`, describing each channel. Columns:

| Column | Description |
| ------ | ----------- |
| `name` | Channel/column name (e.g. `Node_Head_px`, `latency`). |
| `component` | Spatial component (`x`/`y`/`z`, `quat_x…quat_w`, or `n/a`). |
| `type` | BIDS channel type. In practice the pipeline emits `POS`, `ORNT`, `LATENCY`, and `MISC`; other BIDS types (`VEL`, `ACCEL`, `GYRO`, …) are recognized by the schema but are not produced from current Quest/Unity columns. |
| `tracked_point` | The tracked point the channel belongs to (or `n/a` for latency). |
| `units` | Physical units (`m`, `s`, `boolean`, `normalized`, `n/a`, …). |
| `sampling_frequency` | The system's nominal (expected) rate. |
| `reference_frame` | `global` for spatial channels (`POS`/`ORNT`), else `n/a`. |

Channel type, component, and units are inferred from each column's name suffix (e.g. `_px/_py/_pz` → POS in metres; `_qx…_qw` → ORNT quaternion; `latency`/`latency_global` → LATENCY in seconds; face blendshapes → MISC, normalized).

## JSON sidecars

### `motion.json`

Per tracking system. Contains: `TaskName`, `TaskDescription` (from [`system_descriptions`](configuration.md#system_descriptions) or `device.task_description`), `Manufacturer`, `ManufacturersModelName`, `SoftwareVersions`, `TrackingSystemName`, `SamplingFrequency` (expected), `SamplingFrequencyEffective` (measured), `MissingValues`, `MotionChannelCount`, `TrackedPointsCount`, the per-type channel counts (`POSChannelCount`, `ORNTChannelCount`, `LATENCYChannelCount`, `MISCChannelCount`, …), and `DeviceSerialNumber` when present.

`SoftwareVersions` is assembled engine-agnostically from the captured metadata, e.g.:

```
"Unity 6000.0.68f1, OVR Plugin v1.110.0, Horizon OS 2.4, Android OS 14 / API-34 (...)"
```

### `channels.json`

Describes the coordinate system, from [`bids.reference_frame`](configuration.md#bidsreference_frame):

```json
{
  "reference_frame": {
    "Description": "Global VR playspace coordinate system...",
    "Levels": {
      "global": {
        "Description": "...",
        "RotationRule": "left-hand",
        "RotationOrder": "ZXY",
        "SpatialAxes": "RSA"
      }
    }
  }
}
```

### `dataset_description.json`

- **Raw** (root): `Name`, `BIDSVersion`, `DatasetType` (`raw`), `License`, and `Authors` (if any).
- **Derivative**: `Name` suffixed `(preprocessed)`, `DatasetType: derivative`, `GeneratedBy: [{"Name": "ResXR"}]`.

## Events

When the input has an events file and/or custom tables, the pipeline writes a single merged `events.tsv` at the session root (raw tier only):

```
sub-01_ses-01_task-VRtracking_events.tsv
```

It starts with the BIDS-required `onset`, `duration`, then `name`, followed by the union of all [custom-table](input-data.md#custom-tables) columns. Native event rows carry `n/a` in custom columns and vice versa. The sibling `events.json` documents `onset`, `duration`, `name` (with `Levels` listing observed event names), and every custom column for which a schema description exists (`Description`, `Format`, and optional `Units`/`Levels`/`Minimum`/`Maximum`).

## `scans.tsv`

One per session (both tiers). Lists each motion file with its acquisition time:

| Column | Value |
| ------ | ----- |
| `filename` | `motion/<motion filename>` |
| `acq_time` | The recording's UTC start (`YYYY-MM-DDThh:mm:ss.000`), or `n/a` if unknown. |

## `participants.tsv` / `participants.json`

Dataset-level. `participants.tsv` has one row per subject — `participant_id`, `age`, `sex`, `handedness` — drawn from the [session mappings](configuration.md#session_mappings) (`n/a` where unset). `participants.json` documents those columns, including the `sex` levels (`M`/`F`) and `handedness` levels (`R`/`L`/`A`).

## `.bidsignore`

Created at the dataset root (only if absent — never overwritten) to keep non-standard artifacts from tripping BIDS validators. It ignores all `.html`, `.png`, and `.jpg` files and the `.git/` directory dataset-wide (not only the generated reports).

## Conventions worth knowing

- **LF line endings** are forced on all generated TSV/JSON files, so output is byte-reproducible across operating systems.
- **Full float precision** is preserved in `motion.tsv` (no rounding).
- **`sourcedata/`** is a verbatim copy of the input session folder (including the `CustomTables/` subfolder), so the original recording always travels with the dataset. It is not re-copied if already present unless `output.overwrite` is set.
