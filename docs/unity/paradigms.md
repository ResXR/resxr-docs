---
icon: lucide/layout-grid
title: Paradigms
description: The three built-in experiments and the exact fields used to configure them.
---

# Paradigms

The template ships three complete experiments under `Assets/ResXR/Demo Experiments/`. Each is a self-contained reference implementation of the [flow model](architecture.md#the-flow-session-task-trial): it provides its own `*_SessionManager`, `*_TaskManager`, `*_TrialManager`, and a `*_SceneReferencer`, plus the data classes it logs. You configure a paradigm by editing fields in the Inspector; you extend one by copying its scripts. New to the Session → Task → Trial structure? Read [Architecture](architecture.md#the-flow-session-task-trial) first.

!!! info "The SceneReferencer pattern"
    Each paradigm has a `*_SceneReferencer` component (a singleton you place once in the scene) holding public references to every scene object the flow needs — instruction panels, position marks, interactive objects, and timing values. The flow managers read these through `XSceneReferencer.Instance.<field>` instead of holding their own scene links, so all wiring lives in one place. To re-point an experiment at different objects, you edit the SceneReferencer, not the managers.

## Binary Choice

A two-alternative forced-choice task. Two images appear in front of the participant; they reach out and touch one with their hand. The chosen side, the named stimulus, the hand used, and the reaction time are logged per trial.

**How a choice is detected.** Each option is a `Choice` component with a `SpriteRenderer` and a trigger collider. `ChoicesManager` shows both options and races their `WaitForTouch()` calls with `UniTask.WhenAny`; the first to be entered by a collider tagged `Toucher` wins. The hand is resolved from the touching collider's `HandCollider` (Left or Right).

**Trial structure.** Each task loads a folder of stimulus sprites and pairs them up; one trial is generated per pair, so the trial count is `floor(spriteCount / 2)`. A fixation cross is shown between stimuli.

Configure it on these components:

| Component | Field | Meaning |
| --------- | ----- | ------- |
| `BinaryChoice_Task` (one per task, in `BinaryChoice_TasksConfiguration.tasks[]`) | `taskName` | Task label, written into the logged events and tables. |
| | `stimuliOrder` | `RandomOrder` (shuffled) or `FixedOrder` (alphabetical by sprite name). |
| | `stimuliFolderPath` | A path **inside a `Resources/` folder** (default `BinaryChoice/StimuliPairs`). Images must be imported as single Sprites; they are loaded with `Resources.LoadAll<Sprite>`. |
| | `taskInstructions` | The `InstructionsPanel` shown before the task. |
| `BinaryChoice_SceneReferencer` | `SecondsBetweenStimuli` | Fixation-cross duration between trials (default `0.5`). |
| | `instructionsDisplayTime` | How long task instructions stay up (default `3`). |
| | `fixationCross`, `choicesManager`, `generalInstructions`, `endInstructions` | Scene object references. |
| `ChoicesManager` | `choiceA`, `choiceB` | The two `Choice` slots. |

**Logged data.** Generic event markers (`task_start:…`, `trial_start:…`, `fixation`, `session_start`/`session_end`) plus two custom tables: `ChoiceEvents` (one row per trial — `Task`, `Trial`, `OptionAName`, `OptionBName`, `Choice`, `ChosenOption`, `HandUsed`, `ReactionTime`, `displayTime`, `ChoiceTime`) and `StimulusBounds` (one row per slot — the renderer and collider center/size of each option). See [Data Output](data-output.md#custom-tables).

Scene: `Binary Choice Experiment.unity`.

## Maze Navigation

The participant stands in a physical-scale maze and walks to reach a coin. Between trials the maze rotates 180° so the same physical space yields a new route.

**How a trial ends.** The trial awaits `Coin.WaitForCoinPickup()`, which resolves when the coin is touched. The coin debounces pickups (a one-second `_acceptPickUps` window) so that multiple fingers contacting it in the same instant count once, then plays its hide/press animation and a sound.

**Trial structure.** `Maze_Task.numOfTrials` (default 5) sets the trials per task; the trials carry no per-trial parameters. Before each trial a "Trial #N" panel is shown. Between trials (but not after the last), the view fades to black, `Maze.Rotate180Degrees()` flips the maze around its vertical axis, and the view fades back.

| Component | Field | Meaning |
| --------- | ----- | ------- |
| `Maze_Task` (in `Maze_SessionManager._tasks[]`) | `numOfTrials` | Number of trials in the task (default `5`). |
| `Maze_SceneReferencer` | `startingPositionMark` | Where the participant begins each trial (`PlayerPositionMark`). |
| | `coin`, `maze` | The target coin and the rotatable maze. |
| | `trialStartPanel`, `generalInstructions`, `endInstructions` | Instruction panels. |
| | `trialNumberPanelVisibleDuration` | "Trial #N" display time (default `3`). |
| `Coin` | `_facingForward` | Orientation of the coin mesh. |

**Logged data.** Event markers (`trial_start:…`, `coin_pickup:…`, `maze_rotation:state=…`, `player_at_start_zone`, …) plus the `MazeTrialData` custom table (one row per trial — `Task`, `Trial`, `TrialName`, `MazeRotatedAtStart`, the coin position `CoinX/Y/Z`, and the start-zone position `StartZoneX/Y/Z`). It records the same `Task`/`Trial`/`TrialName` columns as the shared `TrialsData` table, plus the Maze-specific columns.

Scene: `Maze Experiment.unity`.

## Museum Viewing

A gallery experience with two task types, selected per task by `Museum_Task.taskType`:

- **FreeExploration** — the participant walks the gallery for `durationInSeconds`. There is no interaction; head, position, and gaze are captured passively by the recorder. Implemented as a single trial that waits out the duration.
- **ImagesRating** — a sequence of images is presented one at a time, each with a slider. The participant sets the slider and confirms; the raw and normalized ratings are logged. One trial is generated per image, in array order.

Before the rating task the player is repositioned to a marked spot and shown rating instructions. The Inspector drawer (`MuseumTaskDrawer`) hides `durationInSeconds` unless the task type is `FreeExploration`.

| Component | Field | Meaning |
| --------- | ----- | ------- |
| `Museum_Task` (in `Museum_SessionManager._tasks[]`) | `taskType` | `FreeExploration` or `ImagesRating`. |
| | `durationInSeconds` | Free-exploration length (shown only for that type). |
| `Museum_SceneReferencer` | `artworks` (`Renderer[]`) | The gallery pieces. Each GameObject's **name becomes the `ArtworkName`** in the logged bounds, so give them meaningful, unique names. |
| | `artworkColliders` (`Collider[]`) | Colliders matching `artworks` in order. |
| | `ratingTaskPlayerPositionMark` | Where the participant stands to rate. |
| | `imagesRating`, plus the four instruction panels (`welcomeInstructions`, `endInstructions`, `endOfExplorationInstructions`, `ratingInstructions`) | Scene references. |
| `ImagesRating` | `imagesToRate` (`SpriteRenderer[]`) | The images to present, in order. |
| | `ratingPanel`, `ratingSlider` | The rating UI. The `Slider` exposes `MinValue`, `MaxValue`, `NumOfIntervals`, `AllowContinuousValues`. |

**Logged data.** Event markers (`trial_start:…`, `free_exploration:…`, …) plus three custom tables: `ImageRatings` (per rated image — `Task`, `Trial`, `ImageName`, `RawRating`, `NormalizedRating`; onset is presentation start, duration is deliberation time), `SliderConfig` (one row — the slider's range and interval settings), and `ArtworkBounds` (one row per artwork — name, renderer center/size, Euler rotation, and collider center/size).

Scene: `Museum Experiment.unity`.

## Building your own

The fastest route to a new experiment is to copy the paradigm closest to yours and edit its flow scripts and SceneReferencer, or to start from the empty Flow Management scaffold described in [Architecture](architecture.md#the-flow-session-task-trial). For logging your own per-trial fields, see [Scripting & API → Custom tables](scripting.md#custom-tables-and-events) and [Extending the Template](extending.md).
