---
icon: lucide/code
title: Scripting & API
description: The components and methods you call to log data and drive the player from experiment code.
---

# Scripting & API

This page is the reference for the APIs you call from experiment scripts: logging data through `ResXRDataManager`, reading tracking through `ResXRPlayer`, and the flow hooks you override. All managers are singletons reached through `.Instance`.

## Logging data

`ResXRDataManager` (namespace `ResXRData`) exposes three ways to write to the session, all timestamped on the shared `timeSinceStartup` clock.

### Events

```csharp
void ReportEvent(string name, float onset, float duration)
```

Appends one row to `*_Events.csv`. Use `duration = 0` for an instantaneous marker; for an interval, emit at the end with the measured elapsed time.

```csharp
float t = Time.realtimeSinceStartup;
ResXRDataManager.Instance.ReportEvent("stimulus_onset", t, 0f);
// … later, when the window closes …
ResXRDataManager.Instance.ReportEvent("stimulus", onsetTime, Time.realtimeSinceStartup - onsetTime);
```

### Debug log

```csharp
void LogLineToFile(string line)
```

Appends a free-text row to `*_ResXRDebugLogs.csv` (with onset/duration). Useful for notes you want timestamped alongside the data. The manager also writes here automatically when schema validation fails.

### Custom tables and events

For structured, experiment-specific data, define a class implementing `CustomDataClass` and log instances with:

```csharp
void LogCustom(CustomDataClass data)
void LogCustom(Func<CustomDataClass> make)   // lazy: only constructs if logging is active
```

The class's `onset` and `duration` become the first two columns; each public field becomes a further column, annotated with `[ColumnInfo]` for BIDS metadata. The CSV is named after the class. The recommended pattern is a static `Log(...)` helper on the data class itself:

```csharp
public class TrialsData : CustomDataClass
{
    public float onset { get; }
    public float duration { get; }

    [ColumnInfo("Task label")]                     public string Task;
    [ColumnInfo("Trial index", Format = "index")]  public int Trial;
    [ColumnInfo("Human-readable trial name")]      public string TrialName;

    public static void Log(string task, int trial, string name, float start, float end)
        => ResXRDataManager.Instance.LogCustom(new TrialsData(task, trial, name, start, end));
}
```

`TrialsData` ships as a shared per-trial summary table; `Events` and `ResXRDebugLogs` are built-in tables tagged `[BuiltInTable]` so they land at the session root. See [Data Output → Custom tables](data-output.md#custom-tables) for the file mechanics and `[ColumnInfo]` options.

### Other `ResXRDataManager` members

| Member | Purpose |
| ------ | ------- |
| `string SessionTime { get; }` | The session timestamp used as the file prefix. |
| `string GetOutputDirectory()` | Absolute path of the current session folder. |
| `UniTask StartBodyTrackingAsync(BodyJointSet, BodyTrackingFidelity2)` | Starts Meta body tracking (defaults: full-body skeleton, high fidelity). |
| `event Action<ContinuousSample> OnContinuousSample` | Fires once per tick before the continuous row is written — hook to mirror data live. |
| `event Action<FaceExpressionSample> OnFaceExpressionSample` | Same, for the face row. |
| Inspector: `recordingOptions`, `recordFaceExpressions`, `exportInEditor`, `saveFilePath`, `csvDelimiter` | Configure what and where to record; see [Recording](recording.md). |

## Reading the player

`ResXRPlayer` is the facade over the tracking rig.

| Member | Type | Description |
| ------ | ---- | ----------- |
| `PlayerHead`, `RightHand`, `LeftHand` | `Transform` | Head and hand anchor transforms. |
| `HandLeft`, `HandRight` | `ResXRHand` | The hand components (pinching, finger colliders). |
| `EyeTracker` | `ResXREyeTracker` | Gaze data. |
| `FocusedObject` | `Transform` | The object the combined gaze ray currently hits (a pass-through to `EyeTracker.FocusedObject`). |
| `EyeGazeHitPosition` | `Vector3` | World-space gaze hit point. |
| `IsEyeTrackingEnabled`, `IsFaceTrackingEnabled` | `bool` | Whether those features are active. |
| `OVRFace` | `OVRFaceExpressions` | Meta face-expression source. |
| `ControllersInputManager`, `PinchingInputManager` | — | Input managers for controller and pinch input. |
| `FadeViewToColor(Color, float)` | `UniTask` | Fades the view to a color (used for scene transitions). |
| `RepositionPlayer(PlayerRepositioner)` | `void` | Moves the rig to a marked position. |
| `SetPassthrough(bool)` | `void` | Toggles passthrough (camera) view. |
| `GetHandFingerCollider(HandType, FingerType)` | `Transform` | The collider transform for a given finger. |

### `ResXREyeTracker`

Per-frame gaze, both combined and per-eye. Properties: `FocusedObject`, `EyeGazeHitPosition`, `EyePosition`, `LeftEyeGazeHitPosition`, `RightEyeGazeHitPosition`, `LeftFocusedObject`, `RightFocusedObject`, `HasLeftEyeHit`, `HasRightEyeHit`. The combined ray runs whenever both eyes report confidence ≥ `0.5`; per-eye rays run only when `EnableSeparateEyeRaycasts` is true (the data manager sets this from `Include Separate Eyes Gaze`).

### `ResXRHand`

`HandType` (`Left`, `Right`, `None`, `Any`), `Pincher`, and `PinchManager`. `GetFingerCollider(FingerType)` returns a finger collider; `FingerType` values map to the hand-bone tip indices (`Thumb`, `Index`, `Middle`, `Ring`, `Pinky`). `SetHandVisibility(bool)` shows or hides the hand mesh.

## Driving the flow

Edit the flow managers directly — or copy a paradigm's set of `*_SessionManager` / `*_TaskManager` / `*_TrialManager` files and rename them — and fill in their hooks. (The shipped paradigms take the copy-and-rename route: they are independent classes, not subclasses of the generic managers.) The managers call the hooks in order:

| Manager | Public entry | Hooks you implement |
| ------- | ------------ | ------------------- |
| `SessionManager` | `RunSessionFlow()` | `StartSession()`, `EndSession()`, `BetweenTasksFlow()` |
| `TaskManager` | `RunTaskFlow(Task)` | `StartTask()`, `EndTask()`, `BetweenTrialsFlow()` |
| `TrialManager` | `RunTrialFlow(Trial)` | `StartTrial()`, `EndTrial()` |

Put your `ReportEvent` and `LogCustom` calls in these hooks so logging lines up with the experiment structure. `EndSession()` should end with `Application.Quit()` so the recorder finalizes its files.

## Scene control

`ResXRSceneManager` manages additive scene loading. The public API is `SwitchActiveScene(string)` (unload the current experiment scene and load the named one) and `RestartActiveScene()`, plus the fields `BaseSceneIndex`, `FirstSceneToLoadIndex`, and the read-only `CurrentSceneName`. `LoadActiveScene(int|string)` and `UnloadActiveScene()` exist but are **private** — used internally by `SwitchActiveScene` and at startup — so call `SwitchActiveScene` from experiment code.

## Interaction building blocks

The demos are assembled from reusable interaction components you can reuse in your own scenes — among them `InstructionsPanel` (and `InstructionsPanelWithConfirmation`) for text panels, `GameButton` for pressable buttons, `Slider` for rating input, `PlayerPositionMark` for "stand here" markers, and the `CollisionDetector`/`ToucherDetector` pattern used to detect a hand touching an object. Many expose `await`-able helpers (for example waiting for a button press, a confirmation, or the participant to reach a mark), which keeps trial logic linear and readable. Browse the paradigm scripts under `Assets/ResXR/Demo Experiments/` for usage.
