---
icon: lucide/rocket
---

# Quickstart

This walkthrough takes you from a freshly installed pipeline to a complete BIDS dataset. It assumes you have [installed](installation.md) the package and activated your environment.

The repository ships with **demo data** under `DATA/Demo_Data/` (three sessions — Binary Choice, Maze Navigation, Museum Viewing) and a ready-to-run config at `config/pipeline_config.yaml`, so you can run the full pipeline immediately.

## 1. Point the config at your data

Open `config/pipeline_config.yaml` and edit the input/output paths and the session mapping. The essential parts:

```yaml
input:
  data_dir: DATA                              # root folder that contains your session folders
  continuous_data_pattern: "*_ContinuousData*.csv"
  face_data_pattern: "*_FaceExpressionData*.csv"
  metadata_pattern: "*SessionMetadata.json"
  events_data_pattern: "*_Events.csv"

output:
  bids_root: output                           # where the BIDS dataset is written
  dataset_name: "ResXR VR Tracking Dataset"
  bids_version: "1.10.1"
  task_name: "VRtracking"
  overwrite: false

# Map each source session folder (relative to data_dir) to BIDS subject/session IDs
session_mappings:
  - source_dir: "Demo_Data/Binary/2026.06.10_18-22"
    subject_id: "01"
    session_label: "01"
  - source_dir: "Demo_Data/Maze/2026.06.10_18-32"
    subject_id: "02"
    session_label: "01"
```

!!! tip "Auto-discovery"
    If you leave `session_mappings` empty, the pipeline discovers every subfolder of `data_dir` that contains a metadata file and numbers the subjects automatically (`sub-01`, `sub-02`, …). Explicit mappings give you control over IDs and let you attach `age`, `sex`, and `handedness`.

See the [Configuration Reference](configuration.md) for every available option.

## 2. Validate before running (dry run)

A dry run loads and validates your configuration, then prints the plan — input dir, output dir, number of sessions, and which tracking systems are enabled — **without writing anything**.

```bash
resxr -c config/pipeline_config.yaml --dry-run
```

If the config has an error, this is where you'll catch it.

## 3. Run the pipeline

=== "Command line"

    ```bash
    resxr -c config/pipeline_config.yaml
    ```

    Add `-v` for verbose (DEBUG) logging or `-q` to show errors only.

=== "Python"

    ```python
    import resxr

    resxr.run(config_path="config/pipeline_config.yaml")
    ```

The pipeline processes each session through all six stages. If one session fails, it is logged and the run continues with the rest.

## 4. Inspect the output

After a successful run, `output/` contains a complete BIDS dataset:

```
output/
├── dataset_description.json
├── participants.tsv / participants.json
├── README
├── .bidsignore
├── sub-01/ses-01/
│   ├── sub-01_ses-01_scans.tsv
│   ├── sub-01_ses-01_task-VRtracking_events.tsv / .json
│   ├── 2026.06.10_18-22_report.html          # ← open this in a browser
│   └── motion/
│       ├── sub-01_ses-01_task-VRtracking_tracksys-Head_motion.tsv / .json
│       ├── ...                                  channels.tsv / .json
│       └── ... (Hands, Eyes, Face)
├── sourcedata/sub-01/ses-01/                  # verbatim copy of the raw inputs
└── derivatives/resxr/                         # preprocessed tier
```

Open the `*_report.html` file in a browser to see the [quality report](reports.md): session summary, per-stream statistics, and an interactive timeline of any quality flags.

## Next steps

- Understand the inputs the pipeline expects → [Input Data Format](input-data.md)
- Tune validation, masking, and metadata → [Configuration Reference](configuration.md)
- Learn the output layout → [BIDS Output](bids-output.md)
