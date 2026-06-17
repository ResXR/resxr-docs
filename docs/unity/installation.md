---
icon: lucide/download
title: Installation
description: Unity version, packages, importing the template, and Quest device setup.
---

# Installation

## Requirements

| Requirement | Value |
| ----------- | ----- |
| Unity Editor | **6000.0.68f1** (Unity 6) — the version the project was authored with. Open it with the matching editor to avoid an upgrade prompt. |
| Android Build Support | Required (Quest is an Android device). Install the **Android Build Support** module, including the OpenJDK and Android SDK/NDK tools, from Unity Hub. |
| Target devices | Meta Quest 2, Quest 3, or Quest Pro (standalone). Eye and face tracking require Quest Pro; the template still runs without them. |
| Render pipeline | Universal Render Pipeline (URP), already configured in the project. |

!!! info "Why this exact editor version?"
    A Unity project records the editor version it was last saved with (`ProjectSettings/ProjectVersion.txt`). Opening it with a newer editor triggers an in-place upgrade. Use **6000.0.68f1** for a clean first open.

## Get the project

The repository is a GitHub **template**, so you start from your own copy rather than cloning the original.

1. On the repository page, click **Use this template → Create a new repository**.
2. Clone your new repository to disk.
3. In **Unity Hub**, click **Add → Add project from disk** and select the cloned folder.
4. Open the project. The first open takes several minutes.

## Packages install themselves

The template does not bundle the Meta XR SDK. Its package dependencies are declared in `Packages/manifest.json`, and Unity's Package Manager downloads them automatically on first open. The key dependencies are:

- **Meta XR SDK 78.0.0** — `com.meta.xr.sdk.core`, `.interaction.ovr`, `.platform`, `.haptics`, `.audio`, `.voice`, plus the MR Utility Kit and XR Simulator.
- **OpenXR 1.16.1** (`com.unity.xr.openxr`) — the XR backend.
- **Input System**, **URP**, and **AI Navigation** (used by the Maze demo).

!!! warning "Let the first import finish"
    Wait for package import and the initial script compile to complete before opening a scene. If the editor reports missing types on first open, restart it once so the freshly imported Meta XR packages are picked up.

Three open-source libraries are bundled directly under `Assets/` (vendored, rather than installed as packages) and need no installation: **UniTask** (Unity-friendly `async`/`await`), **NaughtyAttributes** (extra Inspector controls), and **DOTween** (animation). Their licenses are listed in `THIRD_PARTY_NOTICES.md`.

## Quest device setup

To build and run on hardware:

1. **Enable developer mode** on the headset (via the Meta Horizon mobile app) and connect it over USB. Approve the *Allow USB debugging* prompt inside the headset.
2. In Unity, switch the build target to **Android** under **File → Build Profiles** (formerly Build Settings) and confirm your Quest appears as the run device.
3. For eye and face tracking on Quest Pro, enable **Eye Tracking** and **Natural Facial Expressions** in the headset's Movement Tracking settings and accept the in-app permission prompts the first time the app requests them.

You can also iterate without a headset using the **Meta XR Simulator** (included), or by entering Play mode in the editor with the Quest connected via Quest Link.

When you are set up, continue to the [Quickstart](quickstart.md).
