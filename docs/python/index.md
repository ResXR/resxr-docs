---
icon: lucide/git-branch
---

# Python Pipeline

`resxr-python-pipeline` converts the CSV and JSON files recorded by a Unity/Meta Quest session into a [Motion-BIDS](https://bids.neuroimaging.io/) dataset. It is a Python package (≥3.10) exposing a `resxr` command-line entry point and a small programmatic API.

Input is one or more *session folders* (per-frame tracking CSVs + a `SessionMetadata.json`); output is a BIDS dataset containing per-system motion files, channel descriptions, JSON sidecars, merged events, and an HTML quality report.

!!! tip "New here? Start with the basics"
    To get the pipeline running, read [Installation](installation.md) then [Quickstart](quickstart.md) — those two pages are all you need to produce a dataset from the bundled demo data. The remaining pages are reference material you can consult as needed.

!!! info "What is BIDS, in one paragraph?"
    [BIDS](https://bids.neuroimaging.io/) (Brain Imaging Data Structure) is a community standard for organizing research datasets into a predictable folder layout with machine-readable metadata. **Motion-BIDS** is its extension for motion-tracking data. In practice it means the dataset comes out organized the same way every time and is self-describing, so it is easy to share and to open with standard tools. You do **not** need to know BIDS to use the pipeline — it produces the standard layout for you.

## Processing model

Each session is processed independently through six sequential stages:

```mermaid
flowchart LR
    A[Load<br/>CSV + JSON] --> B[Split<br/>by tracking system]
    B --> C[Validate<br/>quality checks]
    C --> D[Preprocess<br/>optional NaN masking]
    D --> E[Export BIDS<br/>motion / channels / sidecars]
    E --> F[Report<br/>HTML quality report]
```

| Stage | Operation | Reference |
| ----- | --------- | --------- |
| 1. Load | Read the continuous, face, events, metadata, and custom-table files. | [Input Data Format](input-data.md) |
| 2. Split | Slice the wide continuous CSV into one stream per tracking system by column prefix. | [Pipeline Stages](pipeline-stages.md#2-split) |
| 3. Validate | Run configured quality checks; attach quality flags to each stream. | [Validation](validation.md) |
| 4. Preprocess | Optionally mask flagged segments as `NaN` (derivative tier only). | [Pipeline Stages](pipeline-stages.md#4-preprocess) |
| 5. Export BIDS | Write motion/channels/sidecar files, events, scans, and dataset files. | [BIDS Output](bids-output.md) |
| 6. Report | Generate an HTML quality report with a flag timeline. | [Quality Reports](reports.md) |

## Definitions

**Tracking system.** One subsystem of the headset — `Head`, `Hands`, `Eyes`, `Face`, `Body`, or `Controllers`. Each maps to a BIDS `tracksys-` entity with its own motion and channels files. Systems are identified from column-name prefixes and can be enabled or disabled individually.

**Output tiers.** Every session is written twice: a **RAW** tier at the dataset root (an exact standardization of the recording, no values changed) and a **DERIVATIVE** tier under `derivatives/resxr/` (the same data after optional quality masking). The raw tier is never modified by preprocessing.

**LATENCY channels.** Internal time columns (`timestamp`, `timeSinceStartup`) are not exported verbatim. At export each stream is passed through `prepare_motion_data`, which converts them into BIDS LATENCY channels — `latency` (seconds from the stream's recording onset) and, when a per-system clock is used, `latency_global` (seconds from the global engine clock onset). See [Pipeline Stages](pipeline-stages.md#latency-channels).

**Software/version capture.** Software and version metadata is captured from any `*version*` key in the session metadata, so a new headset or engine generally requires no code changes. See [Input Data Format](input-data.md#software-and-version-metadata).

## Invocation

```bash
resxr -c config/pipeline_config.yaml          # CLI
```

```python
import resxr
resxr.run(config_path="config/pipeline_config.yaml")   # programmatic
```

All behavior is driven by a single [YAML configuration file](configuration.md).
