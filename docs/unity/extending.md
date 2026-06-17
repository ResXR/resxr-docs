---
icon: lucide/wrench
title: Extending the Template
description: Add a custom collector, run several experiments in one project, and keep frame time healthy.
---

# Extending the Template

## Recording extra data

Before writing a collector, check whether a lighter mechanism covers your need:

- **Track a few scene objects** → add them to **Custom Transforms To Record** on `ResXRDataManager`. Each becomes `Custom_<Name>_px…_qw` columns in the continuous file, no code required. See [Recording](recording.md#subsystems-and-their-toggles).
- **Log structured, event-like data** (trial parameters, ratings, reaction times) → use a [custom table](data-output.md#custom-tables). This is the right tool for anything that is not a per-frame stream.

Write a custom **collector** only when you genuinely need a new per-tick continuous stream.

### Writing a custom collector

A collector implements `IContinuousCollector`:

```csharp
public interface IContinuousCollector : IDisposable
{
    string CollectorName { get; }
    void Configure(ColumnIndex schema, RecordingOptions options);
    void Collect(RowBuffer row, float timeSinceStartup);
}
```

- `Configure` is called once at startup with the built schema, so you can resolve the indices of the columns you will write.
- `Collect` is called every `FixedUpdate`; fill the supplied `RowBuffer` by column name (`row.TrySet("MyColumn", value)`) — leave a cell unset to record a blank for that tick.
- `Dispose` releases any native resources.

Because you are expected to edit the core scripts directly (the template has no plugin layer), wiring a new collector in means two small edits: add your columns to the continuous schema in `SchemaFactories.BuildContinuousDataV2` (in `Core Infrastructure/SchemaBuilder.cs`), and register `new MyCollector()` in the collector list that `ResXRDataManager` builds in `DoInAwake()`, ideally gated on a flag you add to `RecordingOptions`. Follow the existing collectors under `Assets/ResXR/Base Scene/ResXRDataManager/Collectors/` as templates — `SystemStatusCollector` is a compact example.

!!! tip "Keep columns pipeline-friendly"
    The pipeline routes continuous columns to tracking systems by **name prefix** (`Node_Head_*`, `LeftHand_*`, and so on). If you want your columns to join an existing system, match its prefix; if you want a new system, the pipeline can be configured to recognize it. See [Input Data Format → column mapping](../python/input-data.md#how-columns-map-to-tracking-systems).

## Running several experiments in one project

A single project can hold many experiments. Each experiment is its own scene containing its own `*_SceneReferencer` and flow managers, and all of them sit on top of the same Base Scene.

- Add the Base Scene **and** every experiment scene to the build (the Base Scene first/active).
- Select which experiment loads at startup with `FirstSceneToLoadIndex`, or switch at runtime with `ResXRSceneManager.Instance.SwitchActiveScene("Scene Name")`.
- To keep two experiments' flow scripts from clashing, give each its own prefixed copies of the managers — this is exactly what the demos do (`BinaryChoice_SessionManager`, `Maze_TaskManager`, `Museum_TrialManager`, …). The prefix is only a naming convention to let the classes coexist; each is an independent implementation.

Recording is unaffected: `ResXRDataManager` lives in the Base Scene and records continuously across scene switches, so one session can span multiple experiment scenes if you load them in sequence.

## Performance notes

Every enabled subsystem runs its collector on **every physics tick**, and each row is flushed to disk immediately. That makes recording crash-safe, but it means cost scales with both the number of subsystems and the tick rate.

- **Enable only what you need.** Hands and body produce wide schemas (every bone/joint), and **per-eye gaze** (`Include Separate Eyes Gaze`) triples the eye raycasts per frame (three rays instead of one). Leave subsystems off if your study does not use them.
- **Mind the Fixed Timestep.** Halving it doubles the rows, the CPU work, and the file size. 100 Hz (`0.01` s) is a sensible default; raise it only with reason. Set it in **Project Settings → Time → Fixed Timestep**.
- **Immediate flushing trades throughput for safety.** Writing and flushing each row guarantees you never lose more than one tick, at the cost of per-tick disk I/O. This is usually the right trade for research data.
- **`Include Performance` is reserved.** `OVRPerformanceCollector` returns empty data on the OpenXR backend, so leave it off unless you switch backends and need motion-to-photon latency.

A practical baseline for a Quest Pro eye/face study is **Nodes + Eyes + Gaze + Face** at 100 Hz, adding **Hands** or **Body** only when the task requires them.
