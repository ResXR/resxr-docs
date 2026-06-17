---
icon: lucide/rocket
title: Quickstart
description: Open the Base Scene, run a built-in paradigm, and find the recorded files.
---

# Quickstart

This walkthrough runs one of the built-in experiments in the editor and shows you where the recording lands. It assumes you have [installed](installation.md) the project and let the first import finish.

## 1. Open the Base Scene

Open `Assets/ResXR/Base Scene/Base Scene.unity`. This is the persistent scene that holds `ResXRPlayer` and `ResXRDataManager`; every experiment runs on top of it.

!!! note "Two Base Scene variants"
    There is also `Base Scene With Meta Interactions.unity`, which adds Meta's Interaction SDK rig. Use the plain `Base Scene.unity` for the demos unless an experiment needs Meta interactions.

## 2. Choose which experiment loads

The Base Scene contains a **SceneManager** GameObject (the `ResXRSceneManager` component). It additively loads one experiment scene at startup, chosen by **build index** — the position of a scene in the project's build list (**File → Build Profiles**), counting from 0:

- `BaseSceneIndex` — the Base Scene's own index (default `0`).
- `FirstSceneToLoadIndex` — the experiment scene to load on start (default `1`).

Open **File → Build Profiles** and make sure both the Base Scene and the experiment you want are in the scene list and **enabled** (checked), with the Base Scene first. The shipped project does not have every demo scene added and enabled, so you may need to add the one you want: open it, then use **Add Open Scenes** (or drag it into the list). The demo scene files are:

| Paradigm | Scene |
| -------- | ----- |
| Binary Choice | `Binary Choice Experiment.unity` |
| Maze Navigation | `Maze Experiment.unity` |
| Museum Viewing | `Museum Experiment.unity` |

Set `FirstSceneToLoadIndex` to the build index of your experiment scene (and `BaseSceneIndex` to the Base Scene's), counting only enabled scenes.

## 3. Run it

Press **Play** in the editor (with the Quest connected via Quest Link or the Meta XR Simulator running), or build to the headset with **File → Build And Run** while the Android target is selected.

The experiment plays its flow: instructions, then trials. When the session's `EndSession()` (see [Architecture → The flow](architecture.md#the-flow-session-task-trial)) runs it calls `Application.Quit()`, which flushes and closes all files. In the editor, stopping Play mode triggers the same cleanup.

!!! warning "Always end with a clean quit"
    File finalization (flushing the last CSV rows and writing the custom-tables sidecar) happens when `ResXRDataManager` is destroyed by a clean `Application.Quit()`. If the OS kills the app instead — for example the participant takes the headset off mid-session — the final rows and the sidecar JSON may be lost.

## 4. Find the recorded files

Each run creates one session folder named with a UTC timestamp, `yyyy.MM.dd_HH-mm` (for example `2026.06.16_09-55`). The full path is printed to the console at startup. Where it is written depends on where you ran:

=== "On the headset (build)"

    `Application.persistentDataPath`, which on Quest is:

    ```
    /sdcard/Android/data/<your.bundle.id>/files/<sessionTime>/
    ```

    Pull the folder off the device with `adb pull` or a file manager after the session.

=== "In the editor"

    A temporary folder, `…/Temp/ResXR_EditorLogs/<sessionTime>/`, unless you tick **Export In Editor** on `ResXRDataManager` and set a **Save File Path**, in which case it writes there.

Inside the session folder you will find `*_ContinuousData.csv`, `*_SessionMetadata.json`, `*_Events.csv`, and a `*_CustomTables/` subfolder. See [Data Output](data-output.md) for the full layout, then feed the folder to the [Python pipeline](../python/quickstart.md).

## Next steps

- Configure the demo you ran → [Paradigms](paradigms.md)
- Choose what gets recorded → [Recording](recording.md)
- Understand the scene and flow structure → [Architecture](architecture.md)
