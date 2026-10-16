---
icon: lucide/bar-chart-3
---

# Quality Reports

When [`report.enabled`](configuration.md#report) is true, the pipeline writes one HTML quality report per session.

The file is written to `<session_dir>/<session_id>_report.html` by default, or to `report.output_dir` if set. It is a single standalone HTML file: plotly.js is embedded in it, and the page links no external script or style sheet, so it opens without an internet connection.

!!! note "Reports are not BIDS files"
    The report is a convenience artifact, not part of the BIDS spec, so it is listed in `.bidsignore` and excluded from BIDS validation.

## What the report contains

**Header — session metadata.** Session id, subject, session label, recording start (UTC), UTC offset, device manufacturer/model, platform, build id, sampling mode, fixed delta time, schema revision, the list of **software versions** (engine-agnostic, from the captured `*version*` keys), and which tracking systems were enabled in the recording.

**Summary.** Total duration and the number of warning-level flags for the session, and, when the session has hand-joint channels, the frame detected for each hand's joints (see [BIDS Output](bids-output.md#channelstsv)).

**Checks not run.** Shown only when an enabled check produced no result for a stream: a table with the check, the stream and the reason (see [Checks not run](validation.md#checks-not-run)). The reports of sessions in which every check ran have no such section.

**Per-stream statistics.** A table with, per stream: row count, channel count, expected vs. effective sampling frequency, overall NaN %, and warning count. Each stream expands to a detailed per-column table (count, NaN %, mean, median, std, min, and the 5/25/75/95 percentiles + max).

!!! tip "Statistics come from `stats_summary`"
    The per-stream and per-column statistics tables are populated by the [`stats_summary` check](validation.md#stats_summary). If that check isn't in `enabled_checks`, those tables will be empty.

**Timeline.** Shown when the session has at least one flag: an interactive [Plotly](https://plotly.com/python/) figure with two stacked rows:

- An **events** row (when the session has events) — duration events as bars, instantaneous events as diamond markers, each with a dashed vertical guide line.
- A **quality-flags** row — one horizontal bar per flagged segment, laid out per stream, colored by check and grouped in the legend by check and sub-group. Hovering a bar shows the check, stream, time range, and message.

**Quality flags table.** Every flag with its check, stream, group, severity, start, end, duration, and message.

## A single, onset-relative timeline

Different streams can use different per-system clocks, and recordings begin with some zero-padding before the first real sample. To make everything comparable, the report converts all flag times to one shared timeline:

- The **global onset** is the earliest of the streams' first non-zero `timeSinceStartup` values (among the streams that keep a separate `timeSinceStartup` column, that is, those with a per-system clock).
- Every flag's start and end are expressed **relative to that onset**, so the timeline starts at `0`.

The [LATENCY channels](pipeline-stages.md#latency-channels) and the event onsets in the BIDS output count from the session's time zero, the first valid `timeSinceStartup` of the continuous data file. The two are the same value when a stream from the continuous file keeps a separate `timeSinceStartup` column and no stream's `timeSinceStartup` starts earlier (as in the demo sessions with the shipped config, where the face files start at the same value); the report and the data then agree on when the recording "started."
