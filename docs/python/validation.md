---
icon: lucide/shield-check
---

# Validation & Quality Checks

During the [validate stage](pipeline-stages.md#3-validate), the pipeline runs a configurable set of **quality checks** over each tracking stream. Each check inspects the data and emits zero or more **quality flags** — time segments where something looks wrong. Flags appear in the [HTML report](reports.md) and can optionally drive [NaN masking](pipeline-stages.md#4-preprocess) of the derivative tier.

## How checks run

Checks are registered in a global registry and selected by name via [`validation.enabled_checks`](configuration.md#validation). For each stream, the registry runs the enabled checks **in the order listed** and collects their flags.

- **Per-stream checks** (`required_streams = None`) run independently on every stream.
- **Multi-stream checks** declare `required_streams = [TrackingSystem.X, ...]`. They run **once** (when the first required stream is processed) and access the others via `session.get_stream(system)`. If any required stream is missing, the check is skipped.
- A check that **raises an exception** is caught, recorded as a *failed check*, and skipped. The run continues, but a warning notes that the derivative tier may then contain unmasked bad segments.

Unknown names in `enabled_checks` are silently ignored.

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

Flags are built from a boolean mask over the timestamps; contiguous `True` runs become one flag each. The mask is automatically restricted to the recording window (onset → last valid sample), so leading/trailing zero-padding never produces spurious flags.

!!! info "Times are in the global clock"
    Flag bounds are stored in `timeSinceStartup` space so that masking and the report line up across streams that use different per-system clocks. The report shifts them to a shared onset-relative timeline.

---

## Built-in checks

The demo config enables: `hands_tracking_loss`, `sampling_rate`, `stats_summary`, `eyes_closed`, `clock_dropout`.

### `hands_tracking_loss`

Detects loss of hand tracking using validity flag columns.

- **Scope:** Hands stream (`required_streams = [HANDS]`).
- **Logic:** For each configured hand validity column, flags samples where the value is `0` or `NaN` (tracking not held).
- **Settings:** `tracking_flags` — a map of hand → validity columns. Default:
  ```yaml
  tracking_flags:
    left_hand:  ["LeftHand_Status_HandTracked"]
    right_hand: ["RightHand_Status_HandTracked"]
  ```
- **Emits:** `severity = warning`, `mask = true`, `group_name` = the hand, `target_columns` = the specific validity column (so only that column is masked, not the whole row).

### `sampling_rate`

Validates timing consistency. Purely informational — its flags are **not** maskable.

- **Scope:** every stream (`required_streams = None`).
- **Logic:** two independent tests over the recording window:
    1. **Rate mismatch** — flags if `|effective − expected| / expected` exceeds `sampling_rate_tolerance` (default `0.10`).
    2. **Irregular sampling** — flags if the coefficient of variation of inter-sample intervals exceeds `sampling_cv_threshold` (default `0.50`).
- **Emits:** `severity = warning`, `mask = false`, spanning the whole valid window.

### `eyes_closed`

Detects periods where both eyes are closed, from the **Face** stream's blendshapes, and propagates them onto the **Eyes** stream.

- **Scope:** Face stream (`required_streams = [FACE]`); Eyes is used opportunistically if present.
- **Logic:** flags samples where both `Eyes_Closed_L` and `Eyes_Closed_R` are ≥ `eyes_closed_threshold` (default `0.9`). Segments shorter than `eyes_closed_min_duration` (default `0.1` s) are dropped when `eyes_closed_use_min_duration` is true. Kept segments are then mapped onto the Eyes stream's timeline.
- **Emits:** `severity = info`, `mask = true`, `group_name = "both_eyes"`. On Face the flag targets the two `Eyes_Closed_*` columns; on Eyes it targets the whole row.

### `clock_dropout`

Detects per-system clock dropouts — moments where a stream's own clock resets to `0` mid-recording while the global clock keeps ticking, which would yield invalid latencies.

- **Scope:** every stream (`required_streams = None`).
- **Logic:** finds contiguous zero/`NaN` blocks in the stream's `timestamp` column that fall **strictly inside** the recording window (leading/trailing zeros are excluded). Flag bounds are recorded in global-clock units so masking lines up.
- **Emits:** `severity = warning`, `mask = true`, `target_columns` empty (masks the whole row).

### `stats_summary`

Computes per-column descriptive statistics for each stream and stores them on the stream for the [report](reports.md). It emits **no flags**.

- **Scope:** every stream.
- **Produces:** per-column `count`, `nan_count`, `nan_pct`, `mean`, `median`, `std`, `min`, `p5`, `p25`, `p75`, `p95`, `max`, plus stream-level `row_count`, `column_count`, and overall `nan_pct`. Time columns are excluded.

!!! tip "Enable `stats_summary` for richer reports"
    The per-stream statistics tables in the HTML report are populated by this check. If it isn't enabled, those tables are empty.

### Summary

| Check | Scope | Maskable | Severity | Detects |
| ----- | ----- | :------: | -------- | ------- |
| `hands_tracking_loss` | Hands | ✅ | warning | Hand tracking dropouts |
| `sampling_rate` | all | ❌ | warning | Rate mismatch / irregular sampling |
| `eyes_closed` | Face → Eyes | ✅ | info | Both eyes closed |
| `clock_dropout` | all | ✅ | warning | Per-system clock resets mid-recording |
| `stats_summary` | all | — | — | (statistics only, no flags) |

---

## Column groups

Many checks can be scoped to **named groups of columns** rather than the whole stream — for example flagging only the left-hand columns when the left hand loses tracking. Groups are defined under [`validation.column_groups`](configuration.md#validationcolumn_groups) and assigned to checks via `settings.check_column_groups`:

```yaml
validation:
  column_groups:
    - name: "Left Hand"
      columns: [Left_XRHand_Wrist_x, Left_XRHand_Wrist_y, Left_XRHand_Wrist_z]
    - name: "Right Hand"
      columns: [Right_XRHand_Wrist_x, Right_XRHand_Wrist_y, Right_XRHand_Wrist_z]
  settings:
    check_column_groups:
      hands_tracking_loss: ["Left Hand", "Right Hand"]
```

Inside a check, `config.get_column_groups(self.name, default_columns=...)` returns the assigned groups. If none are configured, it returns a single default group containing `default_columns`. Referencing a group name that doesn't exist raises a `ConfigurationError`.

---

## Writing a custom check

A check is any object with `name`, `description`, `required_streams`, and a `__call__` returning a list of `QualityFlag`s. Three steps:

**1. Create the check** in `src/resxr/validation/checks/`:

```python
from resxr.core.session import QualityFlag, Session, TrackingStream
from resxr.core.config import ValidationConfig
from resxr.validation.registry import register_check


class MyCheck:
    name = "my_check"
    description = "Short description"
    required_streams = None   # None = per-stream; or [TrackingSystem.HANDS, ...] for multi-stream

    def __call__(self, stream: TrackingStream, session: Session, config: ValidationConfig):
        df = stream.data

        # Resolve this check's column groups (falls back to all columns)
        groups = config.get_column_groups(
            self.name,
            default_columns=[c for c in df.columns if c != "timestamp"],
        )

        flags: list[QualityFlag] = []
        for group in groups:
            # group.name, group.columns, group.description
            ...  # build a boolean mask, then QualityFlag.from_mask(...)
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

!!! tip "Building flags from a mask"
    Use `QualityFlag.from_mask(timestamps, boolean_mask, check_name, system, severity, message, should_mask=..., group_name=..., target_columns=...)` to turn a boolean array into contiguous-segment flags. Pass the stream's `timeSinceStartup` values as `timestamps` so flag times align with masking and the report.

**Multi-stream checks:** set `required_streams = [TrackingSystem.X, TrackingSystem.Y]`. The check runs once (on the first required stream) and reads the others via `session.get_stream(system)`. Use this when a check needs more than one system's data — as `eyes_closed` does with Face and Eyes.
