---
icon: lucide/code
---

# Python API Reference

Beyond the [CLI](cli.md), the pipeline can be driven and inspected from Python. This page covers the public surface exported from the `resxr` package.

## Public exports

```python
from resxr import (
    run,             # run the full pipeline
    PipelineConfig,  # the validated configuration model
    Session,         # per-recording state object
    SessionMetadata, # parsed session metadata
    TrackingStream,  # one tracking system's data
    QualityFlag,     # a detected quality issue
    TrackingSystem,  # enum of tracking systems
    __version__,
)
```

## `run`

```python
def run(config_path: str = "config/pipeline_config.yaml") -> None
```

Runs the complete pipeline for every session named in the config (or auto-discovered). This is the same code path as the `resxr` CLI.

```python
import resxr

resxr.run(config_path="config/pipeline_config.yaml")
```

It returns `None`; the result is the BIDS dataset written to `output.bids_root`. Individual session failures are logged and skipped.

## `PipelineConfig`

The validated configuration. Load it from YAML to inspect or build on:

```python
from resxr import PipelineConfig

config = PipelineConfig.from_yaml("config/pipeline_config.yaml")
print(config.input.data_dir)
print(config.output.bids_root)
print([name for name, sys in config.systems.items() if sys.enabled])
```

`from_yaml(path)` raises a `ConfigurationError` if the file is missing, unparseable, empty, or fails validation. See the [Configuration Reference](configuration.md) for every field. Helpers include `is_system_enabled(name)` and, on the `validation` sub-config, `get_column_groups(check_name, default_columns=...)`.

## Core data structures

These mirror the objects that flow through the [pipeline stages](pipeline-stages.md).

### `TrackingSystem`

An enum of the supported systems: `HEAD`, `HANDS`, `EYES`, `FACE`, `BODY`, `CONTROLLERS`. Each member's `.value` is the BIDS `tracksys-` label (`"Head"`, `"Hands"`, …).

### `Session`

The per-recording carrier. Key attributes: `session_id`, `subject_id`, `session_label`, `metadata`, `streams` (`dict[TrackingSystem, TrackingStream]`), the `raw_*_data` frames, `custom_tables`/`custom_tables_data`, and `merged_events_data`. Useful members:

| Member | Returns |
| ------ | ------- |
| `get_stream(system)` | The `TrackingStream` for a system, or `None`. |
| `has_stream(system)` | Whether a system is present. |
| `all_flags` | All quality flags across streams, sorted by start time. |
| `total_duration_seconds` | Longest stream duration. |
| `total_warning_count` / `total_error_count` | Flag tallies. |
| `masked_percentage` | Fraction of time masked (overlap-aware). |

### `TrackingStream`

One system's data. Attributes include `system`, `data` (raw), `clean_data` (post-masking), `quality_flags`, `sampling_frequency` (expected), `sampling_frequency_effective` (measured), and `channel_count`. Useful properties: `duration_seconds`, `row_count`, `warning_count`/`error_count`, and `get_output_data()` (returns `clean_data` if set, else `data`).

### `QualityFlag`

A detected issue over a time segment — see [Validation](validation.md#quality-flags) for the full field list and the `from_mask(...)` constructor used by checks.

### `SessionMetadata`

The parsed `SessionMetadata.json`: timing (`utc_start`, `fixed_delta_time`), feature flags (`face_enabled`, `hands_enabled`, …), schema counts, device fields, and the engine-agnostic `software_versions` map.

## Loading data without running the pipeline

You can load a single session's raw data directly:

```python
from resxr import PipelineConfig
from resxr.io.readers import load_session

config = PipelineConfig.from_yaml("config/pipeline_config.yaml")
session = load_session("/path/to/session_dir", config.input)
print(session.session_id)
print(session.raw_continuous_data.shape)
```

!!! note "Raw load only"
    `load_session` returns raw data only — streams are not yet split, so `session.streams` is empty and `total_duration_seconds` is `0.0`. Use `run(...)` for the full split → validate → preprocess → export flow.

## Exceptions

All pipeline errors derive from `ResXRError` (importable from `resxr.core`). Catching it handles any pipeline failure:

| Exception | Raised when |
| --------- | ----------- |
| `ResXRError` | Base class for all of the below. |
| `ConfigurationError` | Config is missing, malformed, empty, or invalid. |
| `DataLoadError` | An input file cannot be loaded or parsed. |
| `MissingDataError` | A required file (metadata or continuous CSV) is absent. |
| `ValidationError` | Data fails a critical validation check. |
| `BIDSWriteError` | A BIDS output file cannot be written. |
| `ColumnMappingError` | Column mapping/splitting fails. |

```python
from resxr import run
from resxr.core import ResXRError

try:
    run("config/pipeline_config.yaml")
except ResXRError as exc:
    print(f"Pipeline failed: {exc}")
```
