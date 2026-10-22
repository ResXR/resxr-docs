---
icon: lucide/shield-check
---

# Validation & Quality Checks

During the [validate stage](pipeline-stages.md#3-validate), the pipeline runs a configurable set of **quality checks** over each tracking stream. Each check inspects the data and emits zero or more **quality flags** — time segments where something looks wrong. Flags appear in the [HTML report](reports.md) and can optionally drive [NaN masking](pipeline-stages.md#4-preprocess) of the derivative tier.

## How checks run

Checks are registered in a global registry and selected by name via [`validation.enabled_checks`](configuration.md#validation). For each stream, the registry runs the enabled checks **in the order listed** and collects their flags.

- **Per-stream checks** (`required_streams = None`) run independently on every stream.
- **Multi-stream checks** declare `required_streams = [TrackingSystem.X, ...]`. They run **once** (when the first required stream is processed) and access the others via `session.get_stream(system)`. If a required stream is missing, the check does not run. It is recorded as [not run](#checks-not-run) unless that stream was turned off on purpose: disabled in the config (`systems.<name>.enabled: false`) or set to JSON `false` in its `*_enabled` metadata key.
- A check that **raises an exception** is caught and recorded as not run, with the reason `failed: <error>`. The run continues, but the failure is logged as an error naming the session and noting that the derivative tier may then contain unmasked bad segments.
- The checks that place flags (`sampling_rate`, `hands_tracking_loss`, `clock_dropout`, `eyes_closed`) place them on `timeSinceStartup`, which a stream keeps as a separate column only when it has a per-system clock column ([`preprocessing.alternate_time_columns`](configuration.md#alternate_time_columns)). On a stream without one, such a check logs an error, is recorded as not run, and raises no flags.

An unknown name in `enabled_checks` is a [configuration error](configuration.md#configuration-errors) that lists the valid names.

### Checks not run

An enabled check that produced no result for a stream is recorded with the check, the stream and the reason: its stream is absent although it was expected (the reason says why, e.g. `the session has no Face stream (metadata: face_enabled is true; no face file was loaded)`), a column it reads is missing or holds values it cannot read, the stream has no separate `timeSinceStartup` column, or the check failed. Each case is logged when it happens and listed in the [run summary](cli.md#run-summary) (which is then a warning), in the session's `checks_not_run` in the [`RunResult`](api.md#run), and in the report's "Checks not run" section ([Quality Reports](reports.md#what-the-report-contains)).

A check that did not run does not skip the session: its data are exported, and the exit code stays `0` when every session was processed. With [quality masking](pipeline-stages.md#4-preprocess) on, such a check masks nothing.

## Quality flags

Each flag is a `QualityFlag` with:

| Field | Meaning |
| ----- | ------- |
| `check_name` | The check that produced it. |
| `system` | Which tracking system it applies to. |
| `start_time`, `end_time` | Segment bounds, in `timeSinceStartup` (global clock) seconds. |
| `severity` | `"info"`, `"warning"`, or `"error"`. |
| `message` | Human-readable description. |
| `mask` | Whether the segment is eligible for NaN masking. |
| `group_name` | Optional label (e.g. `"left_hand"`). |
| `target_columns` | Columns the flag applies to. **Empty = the whole row**; non-empty = only those columns. |

Flags are built from a boolean mask over the timestamps; contiguous `True` runs become one flag each. The mask is automatically restricted to the recording window (onset → last valid sample), so leading/trailing zero-padding never produces spurious flags. Rows whose time is zero or not finite are also left out of the mask, so a flag never starts or ends on such a row, and one inside a flagged stretch splits it into two flags.

!!! info "Times are in the global clock"
    Flag bounds are stored in `timeSinceStartup` space so that masking and the report line up across streams that use different per-system clocks. The report shifts them to a shared onset-relative timeline.

---

## Built-in checks

The demo config enables: `hands_tracking_loss`, `sampling_rate`, `stats_summary`, `eyes_closed`, `clock_dropout`.

### `hands_tracking_loss`

Detects loss of hand tracking using validity flag columns.

- **Scope:** Hands stream (`required_streams = [HANDS]`).
- **Logic:** For each configured hand validity column, flags samples where the value is `0`, `false` or blank (tracking not held); `1` and `true` (any case, also as text) count as tracked. A column that is missing, or that holds any other value, gives no flags: the check warns and is recorded as [not run](#checks-not-run) for that hand.
- **Settings:** `tracking_flags` — a map of hand → validity columns. Default:
  ```yaml
  tracking_flags:
    left_hand:  ["LeftHand_Status_HandTracked"]
    right_hand: ["RightHand_Status_HandTracked"]
  ```
- **Emits:** `severity = warning`, `group_name` = the hand, `target_columns` = that hand's measurement columns (the columns with that hand's prefixes, except its status, validity and timing columns), so masking removes that hand's measurements and keeps its status columns; `mask = true` when the hand has such columns.

### `sampling_rate`

Validates timing consistency. Purely informational — its flags are **not** maskable.

- **Scope:** every stream (`required_streams = None`).
- **Logic:** two independent tests on the stream's own clock (its per-system clock column), from the recording onset (the clock's first non-zero value):
    1. **Rate mismatch**: flags if `|effective − expected| / expected` exceeds `sampling_rate_tolerance` (default `0.10`). The effective rate is the number of the stream's samples (its distinct finite, non-zero clock values at or above the onset, `TrackingStream.sample_times()`), minus one, divided by the time they span.
    2. **Irregular sampling**: flags if the coefficient of variation of the intervals between the same samples exceeds `sampling_cv_threshold` (default `0.50`). The intervals between rows on `timeSinceStartup` are not used.
- **Emits:** `severity = warning`, `mask = false`, spanning the whole valid window on `timeSinceStartup`.

### `eyes_closed`

Detects periods where both eyes are closed, from the **Face** stream's blendshapes, and propagates them onto the **Eyes** stream.

- **Scope:** Face stream (`required_streams = [FACE]`); Eyes is used opportunistically if present.
- **Logic:** flags samples where both `Eyes_Closed_L` and `Eyes_Closed_R` are ≥ `eyes_closed_threshold` (default `0.9`). Segments shorter than `eyes_closed_min_duration` (default `0.1` s) are dropped when `eyes_closed_use_min_duration` is true. Kept segments are then mapped onto the Eyes stream's rows by `timeSinceStartup`. If either column is missing from the face file, the check warns and is recorded as [not run](#checks-not-run).
- **Emits:** `severity = info`, `group_name = "both_eyes"`. On Face the flag targets the two `Eyes_Closed_*` columns with `mask = false`, so the Face stream is not masked. On Eyes it targets the stream's position and orientation columns (`POS`/`ORNT`) except `Node_EyeCenter_*`, with `mask = true` when there are such columns.

### `clock_dropout`

Detects per-system clock dropouts — moments where a stream's own clock resets to `0` mid-recording while the global clock keeps ticking, which would yield invalid latencies.

- **Scope:** every stream (`required_streams = None`).
- **Logic:** finds contiguous zero/`NaN` blocks in the stream's `timestamp` column that fall **strictly inside** the recording window (leading/trailing zeros are excluded). Flag bounds are recorded in global-clock units so masking lines up.
- **Emits:** `severity = warning`, `mask = true`, `target_columns` empty (masks the whole row).

### `stats_summary`

Computes per-column descriptive statistics for each stream and stores them on the stream for the [report](reports.md). It emits **no flags**.

- **Scope:** every stream.
- **Produces:** per-column `count`, `nan_count`, `nan_pct`, `mean`, `median`, `std`, `min`, `p5`, `p25`, `p75`, `p95`, `max`, plus stream-level `row_count`, `column_count`, and overall `nan_pct`. Only numeric columns are summarized; `timestamp`, `timeSinceStartup` and `Eyes_Time` are excluded.

!!! tip "Enable `stats_summary` for richer reports"
    The per-stream statistics tables in the HTML report are populated by this check. If it isn't enabled, those tables are empty.

### Summary

| Check | Scope | Maskable | Severity | Detects |
| ----- | ----- | :------: | -------- | ------- |
| `hands_tracking_loss` | Hands | ✅ | warning | Hand tracking dropouts |
| `sampling_rate` | all | ❌ | warning | Rate mismatch / irregular sampling |
| `eyes_closed` | Face → Eyes | ✅ (Eyes only) | info | Both eyes closed |
| `clock_dropout` | all | ✅ | warning | Per-system clock resets mid-recording |
| `stats_summary` | all | — | — | (statistics only, no flags) |

---

## Column groups

A custom check can be scoped to **named groups of columns** rather than the whole stream. No built-in check reads column groups. Groups are defined under [`validation.column_groups`](configuration.md#validationcolumn_groups) and assigned to checks via `settings.check_column_groups`:

```yaml
validation:
  column_groups:
    - name: "Left Hand"
      columns: [Left_XRHand_Wrist_x, Left_XRHand_Wrist_y, Left_XRHand_Wrist_z]
    - name: "Right Hand"
      columns: [Right_XRHand_Wrist_x, Right_XRHand_Wrist_y, Right_XRHand_Wrist_z]
  settings:
    check_column_groups:
      my_check: ["Left Hand", "Right Hand"]
```

Inside a check, `config.get_column_groups(self.name, default_columns=...)` returns the assigned groups, or every group when the check has no entry in `check_column_groups`. If no groups are configured, it returns a single default group containing `default_columns`. A group name in `check_column_groups` that `column_groups` does not define is a [configuration error](configuration.md#configuration-errors) at start-up.

---

## Writing a custom check

A check is any object with `name`, `description`, `required_streams`, and a `__call__` returning a list of `QualityFlag`s. A check that reads keys under `validation.settings` also lists them in `settings_keys`. Three steps:

**1. Create the check** in `src/resxr/validation/checks/`:

```python
from resxr.core.session import QualityFlag, Session, TrackingStream
from resxr.core.config import ValidationConfig
from resxr.validation.registry import register_check


class MyCheck:
    name = "my_check"
    description = "Short description"
    required_streams = None   # None = per-stream; or [TrackingSystem.HANDS, ...] for multi-stream
    settings_keys = ("my_check_threshold",)   # the validation.settings keys this check reads

    def __call__(self, stream: TrackingStream, session: Session, config: ValidationConfig):
        threshold = config.get("my_check_threshold", 0.5)
        df = stream.data

        # Resolve this check's column groups (falls back to all columns)
        groups = config.get_column_groups(
            self.name,
            default_columns=[c for c in df.columns if c != "timestamp"],
        )

        flags: list[QualityFlag] = []
        for group in groups:
            # group.name, group.columns, group.description
            ...  # build a boolean mask (e.g. values above threshold), then QualityFlag.from_mask(...)
        return flags


register_check(MyCheck())   # register an instance
```

**2. Export it** from `checks/__init__.py` so importing the package registers it.

**3. Enable it** in your config:

```yaml
validation:
  enabled_checks:
    - my_check
```

!!! note "Declare the settings keys a check reads"
    A key under `validation.settings` that no registered check lists in its `settings_keys` is a [configuration error](configuration.md#configuration-errors), so a custom check that reads a key with `config.get(key, default)` must list it there. A check without `settings_keys` declares none.

!!! tip "Building flags from a mask"
    Use `QualityFlag.from_mask(timestamps, boolean_mask, check_name, system, severity, message, should_mask=..., group_name=..., target_columns=...)` to turn a boolean array into contiguous-segment flags. Pass the stream's `timeSinceStartup` values as `timestamps` so flag times align with masking and the report. A stream has a separate `timeSinceStartup` column only when it has a per-system clock; `resxr.validation.registry.has_global_clock(self.name, stream, session)` returns `False` for a stream without one, after logging the error and recording the check as not run, as the built-in checks do.

**Multi-stream checks:** set `required_streams = [TrackingSystem.X, TrackingSystem.Y]`. The check runs once (on the first required stream) and reads the others via `session.get_stream(system)`. Use this when a check needs more than one system's data. A missing required stream stops the check, so a stream it only uses when present is not listed: `eyes_closed` lists Face and reads Eyes when the session has it.
