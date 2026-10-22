# Building players

## Unity version

The template targets **Unity 6000.6.6f1**. Any newer Unity 6 release works; the
project will prompt to upgrade. Unity 2022.3 and earlier are not supported.

## From the editor

All build commands live under **Tools → Chosen of the Harvest**:

| Menu item | Result |
|---|---|
| **Create or Repair Bootstrap Scene** | Rebuilds `Assets/ChosenOfTheHarvest/Scenes/ChosenOfTheHarvest.unity` |
| **Build Windows Player** | `Build/ChosenOfTheHarvest.exe` |

`Editor/BuildGame.cs` also exposes `BuildLinux` and `BuildMac` as static
methods; they are not menu items because they require the matching platform
module.

## From the command line

```powershell
$unity = "C:\Program Files\Unity\Hub\Editor\6000.6.6f1\Editor\Unity.exe"

& $unity -batchmode -quit -projectPath "C:\path\to\ChosenOfTheHarvest" `
  -executeMethod ChosenOfTheHarvest.EditorTools.BuildGame.BuildPlayer `
  -logFile "BuildLogs\build.log"
```

| Method | Output |
|---|---|
| `BuildGame.BuildScene` | Rebuilds the scene |
| `BuildGame.BuildPlayer` | `Build/ChosenOfTheHarvest.exe` |
| `BuildGame.BuildLinux` | `Build/Linux/ChosenOfTheHarvest.x86_64` |
| `BuildGame.BuildMac` | `Build/Mac/ChosenOfTheHarvest.app` |

Constants at the top of `BuildGame.cs`:

```csharp
public const string ScenePath = "Assets/ChosenOfTheHarvest/Scenes/ChosenOfTheHarvest.unity";
public const string ProductName = "ChosenOfTheHarvest";
public const string Version = "1.0.0";
```

Change these once and every target follows.

## The helper script

```powershell
.\Tools\build.ps1              # Windows build
.\Tools\build.ps1 -RunSmoke    # build + run the -smoke verification
.\Tools\build.ps1 -Linux       # Linux build
```

Set `UNITY_PATH` to override the editor path. **Logs go to `BuildLogs/`** —
`build_<timestamp>.log`, `latest.log`, `SMOKE_LOG.txt`. The `Build/` folder is
player output only.

## Platform modules

Install modules in Unity Hub → **Installs → Add modules**:

| Target | Module |
|---|---|
| Windows | Windows Build Support (Mono) |
| Linux | Linux Build Support (Mono) |
| macOS | macOS Build Support (Mono) |

The template builds with the **Mono** backend so buyers do not need IL2CPP
toolchains. For release builds, switch to IL2CPP in
**Project Settings → Player → Configuration**.

## Verifying a build

```powershell
.\Tools\build.ps1 -RunSmoke
```

What happens:

1. Unity builds `Build/ChosenOfTheHarvest.exe`.
2. The script launches it with `-smoke`.
3. `SmokeTest.cs` drives the game: audio, weapons, dodge, damage, building,
   production, hauling, companions, needs, beds, jobs, menus, inventory,
   catalog, repairs, interiors, quests, campaign, floor plans, combat, elites,
   shops, puzzles, curses, bosses, omens, descent, victory, meta, ceremonies,
   save/load, pause and localisation.
4. On success the log ends with `SMOKE done` and the script exits `0`.

Logs:

```
BuildLogs/SMOKE_LOG.txt
%USERPROFILE%\AppData\LocalLow\ChosenOfTheHarvest\ChosenOfTheHarvest\Player.log
```

Use this after any code change — it is the fastest way to catch a broken system.

## Build size

A Windows development build is roughly **180 MB**, dominated by
`UnityPlayer.dll` and the Mono runtime. The package you submit to the Asset
Store is the **project**, not the build; the project's `Assets` folder is about
43 MB.

!!! warning "Never submit a build"

    The Asset Store does not accept executables inside a package. Do not include
    `Build/`, `Library/`, `Logs/`, `BuildLogs/` or `UserSettings/` in a
    submission.

## CI notes

- The build is deterministic enough for CI; the smoke test is the regression
  gate.
- Run Unity with `-batchmode -quit -nographics` for headless machines.
- The editor writes its licence state to the machine; a CI runner needs a
  licensed Unity install or a Unity licence server.
- `BuildLogs/` is git-ignored by default.

## Related

- [Asset Store submission](asset-store.md)
- [Troubleshooting](../troubleshooting.md)
- [Project structure](../getting-started/project-structure.md)
