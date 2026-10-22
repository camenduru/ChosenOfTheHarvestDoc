# Installation

This page takes you from the store download to a running project. It assumes
nothing is installed yet.

## 1. Install Unity

You need **Unity 6000.6.6f1** or a newer Unity 6 release.

1. Install **Unity Hub** from <https://unity.com/download>.
2. In Hub, open **Installs → Install Editor → Archive**.
3. Pick **6000.6.6f1** and install it with:
    - **Windows Build Support (Mono)** or **macOS Build Support (Mono)**
    - **Linux Build Support (Mono)** if you target Linux

!!! tip "Module choice"

    The template builds with the **Mono** scripting backend, so you do not need
    the IL2CPP toolchains to run or build it. Add IL2CPP later if you want it
    for release builds.

## 2. Get the project

Download the package from the Unity Asset Store (**My Assets → Download**), or
unzip the project archive you received.

You should end up with a folder containing:

```
Assets/
Packages/
ProjectSettings/
README.md
CHANGELOG.md
Tools/
```

If you received a `.unitypackage` instead, create a new **3D (URP)** project in
Unity Hub first, then import the package into it.

## 3. Add it to Unity Hub

1. Open **Unity Hub → Projects**.
2. Click **Add → Add project from disk**.
3. Select the folder that contains `Assets`, `Packages` and `ProjectSettings`.
4. Open it with **6000.6.6f1**.

!!! warning "Do not create a new project inside the template"

    The folder you select must be the project root — the one that already has
    `ProjectSettings/ProjectVersion.txt`. Hub will show the Unity version next
    to it once it is recognised.

## 4. First import

Unity resolves the package manifest and compiles the scripts. Expect:

- a progress bar for the first import (one to three minutes on a laptop);
- no external downloads beyond the declared Unity packages;
- an empty Console when it finishes.

If the Console shows `error CS…`, the project is being opened with an
unsupported Unity version. Check `ProjectSettings/ProjectVersion.txt`.

## 5. Open the scene and play

1. In the Project window, open
   `Assets/ChosenOfTheHarvest/Scenes/ChosenOfTheHarvest.unity`.
2. Press **Play**.
3. Choose **New game**.

The scene contains a single `Game` object. Everything else — camera, hub, HUD,
companions, audio — is created at runtime by `HarvestGame`.

## 6. Optional: verify the package

If you want proof that the whole game loop works on your machine:

```powershell
.\Tools\build.ps1 -RunSmoke
```

This builds a Windows player, runs the built-in `-smoke` pass, and writes
`BuildLogs/latest.log` plus `BuildLogs/SMOKE_LOG.txt`. A pass ends with
`SMOKE done`.

## Troubleshooting the install

| Symptom | Cause | Fix |
|---|---|---|
| Hub says "Unknown version" | Missing editor version | Install 6000.6.6f1 from the Archive tab |
| Package Manager errors on open | No network access to the Unity registry | Connect once so UPM can resolve the manifest |
| `error CS0234: InputSystem` | Input System package not resolved | **Window → Package Manager → In Project**, reinstall *Input System* |
| Pink materials | URP render pipeline asset not assigned | **Project Settings → Graphics**, set *UniversalRP* as the default pipeline asset |
| Empty scene / no Game object | Opened the wrong scene | Open `Assets/ChosenOfTheHarvest/Scenes/ChosenOfTheHarvest.unity` |
| Console warning about domain reload | Informational | See [Fast Enter Playmode](../architecture/overview.md#static-state-and-fast-enter-playmode) |

## Next

- [Your first run](first-run.md)
- [Project structure](project-structure.md)
- [Controls](controls.md)
