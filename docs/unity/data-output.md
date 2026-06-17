---
icon: lucide/file-output
title: Data Output
description: The files a session writes, how they are named and structured, and the contract with the pipeline.
---

# Data Output

Every run writes one **session folder** containing CSV data files and JSON metadata. This folder is the entire interface to the [Python pipeline](../python/index.md) — there is no other coupling between the two components.

!!! info "The data contract is single-sourced"
    The authoritative, column-by-column specification of these files lives on the pipeline side, in **[Input Data Format](../python/input-data.md)**. This page describes how the template *produces* the files — their names, where they go, how the schema is built, and how to add your own tables. For the exact column names, types, and required-vs-optional rules the pipeline reads, follow the link.

## Folder layout

Files are written under the device's persistent data path, in a subfolder named with the UTC session timestamp `yyyy.MM.dd_HH-mm`:

```
{persistentDataPath}/
└── 2026.06.16_09-55/
    ├── 2026.06.16_09-55_ContinuousData.csv       # per-tick tracking (required)
    ├── 2026.06.16_09-55_FaceExpressionData.csv   # face weights (if face enabled)
    ├── 2026.06.16_09-55_Events.csv               # event markers
    ├── 2026.06.16_09-55_ResXRDebugLogs.csv       # debug / validation log
    ├── 2026.06.16_09-55_SessionMetadata.json     # recording config & timing (required)
    └── 2026.06.16_09-55_CustomTables/
        ├── 2026.06.16_09-55_CustomTables.json    # schema for the tables below
        ├── 2026.06.16_09-55_TrialsData.csv
        ├── 2026.06.16_09-55_ChoiceEvents.csv     # (paradigm-specific)
        └── 2026.06.16_09-55_<ClassName>.csv      # one per custom data class
```

The timestamp prefix is identical for every file in a session. On the headset `{persistentDataPath}` resolves to `/sdcard/Android/data/<bundle.id>/files/`; in the editor it defaults to a temporary folder unless you enable **Export In Editor** (see [Quickstart](quickstart.md#4-find-the-recorded-files)).

## The files

| File | Written when | Contents |
| ---- | ------------ | -------- |
| `*_ContinuousData.csv` | Every physics tick | One row per tick with all enabled continuous subsystems (nodes, eyes, hands, body, system status, custom transforms). Hundreds of columns wide. The time axis is `timeSinceStartup`. |
| `*_FaceExpressionData.csv` | Every physics tick, if face recording is on | Facial-expression weights and region confidence, on the same `timeSinceStartup` clock, in a separate file. |
| `*_Events.csv` | On each `ReportEvent(...)` call | Point or interval event markers. Columns: `onset`, `duration`, `name`. |
| `*_ResXRDebugLogs.csv` | On each `LogLineToFile(...)` call, and automatically on schema-validation errors | Free-text debug log. Columns: `onset`, `duration`, `message`. |
| `*_SessionMetadata.json` | Once, at session start | Recording configuration, reference frames, schema counts, device and timing info. |
| `*_CustomTables/*.csv` | On each `LogCustom(...)` for that class | One CSV per custom data class — your experiment-specific tables. |
| `*_CustomTables/*_CustomTables.json` | Once, at session end | A schema describing each custom table's row count and columns. |

`Events` and `ResXRDebugLogs` are themselves custom data classes, but they are tagged `[BuiltInTable]`, which routes them to the **session root** rather than the `CustomTables/` subfolder.

### How rows are written

The CSV writer writes the header lazily from the schema on the first row, formats numbers with the invariant culture (so the decimal separator is always `.`), writes booleans as `true`/`false`, and leaves a cell **blank** when a value is missing for that tick (never zero-filled — blank means "no data"). Every row is flushed to disk immediately, so an unclean shutdown loses at most the current tick. The delimiter is a comma by default (`csvDelimiter` on `ResXRDataManager`).

## Custom tables

A custom table is any C# class implementing the `CustomDataClass` interface. Its `onset` and `duration` become the first two columns; each subsequent **public field** becomes a column, in declaration order. The CSV file is named after the class (`*_<ClassName>.csv`) and created on first write.

```csharp
public class ChoiceEvents : CustomDataClass
{
    public float onset { get; }
    public float duration { get; }

    [ColumnInfo("Task label")]                 public string Task;
    [ColumnInfo("Trial index", Format = "index")] public int Trial;
    [ColumnInfo("Chosen side", "A:left option", "B:right option")] public string ChosenOption;
    // …
}
```

Each field is annotated with **`[ColumnInfo]`**, which carries the BIDS metadata for that column: a required description, an optional `Format` (one of the 18 BIDS column formats such as `index`, `number`, `boolean`, `label`), optional `Units`, `Minimum`/`Maximum`, and enumerated `Levels` written as `"value:description"`. These annotations are emitted into the `*_CustomTables.json` sidecar, which maps every table to its `RowCount` and a `Columns` object (each with `Description`, `Format`, and any `Units`/`Levels`/`Minimum`/`Maximum`).

!!! note "Annotations are validated"
    An editor check flags any public field on a `CustomDataClass` that is missing a `[ColumnInfo]` after each compile, and the data manager re-checks at session end, writing errors to the console and to `*_ResXRDebugLogs.csv`. If two tables share a column name, their `[ColumnInfo]` must match — the pipeline merges same-named columns across tables.

For the code-level API to log events and tables, see [Scripting & API](scripting.md#custom-tables-and-events). For how the pipeline consumes all of this, see [Input Data Format](../python/input-data.md) and [BIDS Output → Events](../python/bids-output.md#events).

## Session metadata

`*_SessionMetadata.json` is written once at startup and never changed. It records: the session id and UTC start time; the Unity, Meta XR runtime, and Horizon OS versions; the sampling mode and `fixedDeltaTime`; which subsystems were enabled and the schema counts (hand bones, body joints, face expressions); the two [reference frames](recording.md#reference-frames) and rotation conventions; and device fields. The pipeline reads it to decide which tracking systems to split out and how to label channels; see [Input Data Format → Session metadata](../python/input-data.md#session-metadata).
