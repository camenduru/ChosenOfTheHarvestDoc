# Troubleshooting

## First, find the logs

| Log | Location |
|---|---|
| Editor | **Console** window, or `Editor.log` in the Unity log folder |
| Build | `BuildLogs/latest.log` |
| Smoke test | `BuildLogs/SMOKE_LOG.txt` |
| Player | `%USERPROFILE%\AppData\LocalLow\ChosenOfTheHarvest\ChosenOfTheHarvest\Player.log` |
| macOS player | `~/Library/Logs/ChosenOfTheHarvest/ChosenOfTheHarvest/Player.log` |
| Linux player | `~/.config/unity3d/ChosenOfTheHarvest/ChosenOfTheHarvest/Player.log` |

## Import and compile

### `Scripts have compiler errors` on first open

1. Confirm `ProjectSettings/ProjectVersion.txt` says `6000.6.6f1`.
2. Open **Window → Package Manager** and check that *Input System*, *uGUI* and
   *Universal RP* are installed.
3. Delete `Library/ScriptAssemblies` and `Library/Bee`, then re-open the project.

If the errors mention a missing `UnityEngine.InputSystem`, the Input System
package did not resolve — see below.

### Package Manager cannot resolve a package

The project needs network access to `packages.unity.com` once. If you are offline:

1. Open the project on a connected machine.
2. Copy the resolved `Library/PackageCache` and `Packages/packages-lock.json`.
3. Or vendor the packages into `Packages/`.

### `Multiple Unity instances cannot open the same project`

A previous batch-mode Unity process is still running and holding the lock. Close
Unity, then end any lingering `Unity.exe` process:

```powershell
Get-Process Unity -ErrorAction SilentlyContinue | Stop-Process -Force
```

### Text in the HUD is missing glyphs

`HarvestHud` uses Unity's built-in `LegacyRuntime.ttf` and falls back to an OS
font. If a language needs glyphs the built-in font lacks, supply a font asset:

1. Import a TTF/OTF.
2. Assign it in `HarvestHud.UiFont` (replace the `GetBuiltinResource` call).
3. Confirm it covers the glyphs in your `en.json` replacement.

## Rendering

### Pink / magenta everything

URP is not assigned. Set the pipeline asset:

- **Project Settings → Graphics → Default Render Pipeline** →
  `Assets/ChosenOfTheHarvest/Settings/UniversalRP.asset`
- **Project Settings → Quality → Render Pipeline Asset** → the same file

### Sprites are huge or tiny

`ArtImporter` sets characters to 400 px/unit and everything else to 100 px/unit.
If you imported art outside that folder, set **Pixels Per Unit** manually to
match.

### Sprites flicker or overlap incorrectly

The template sorts sprites by world Z (`Stage3D.SortOrder`). Custom sprites with
unusual pivots can fight for the same order. Adjust the pivot to bottom-centre or
add a small bias.

## Gameplay runtime

### `NullReferenceException` in `HarvestGame.Instance`

Something ran before the bootstrap. Code that touches `HarvestGame.Instance`
must run after `HarvestGame.Awake`. If you add a system, start it from
`HarvestGame`, not from `Awake` of an unrelated object.

### Ceremonies do nothing

The altar has a cooldown. The toast `The altar is resting (Ns)` tells you how
long is left. Also check `grace`, `food` and `pearls` meet the ceremony cost.

### A companion is stuck

The follower uses `NavGrid`, which is rebuilt when the hub changes. If a
companion is stuck after you place a building, pick the building up and put it
back, or reload the save — both rebuild the nav grid.

### Building placement will not confirm

Placement validates against `RoomCollision` and the hub bounds. Move the ghost
until it turns valid; `Esc` cancels.

### Save fails

`HarvestSaveSystem` writes to `Application.persistentDataPath`. A failure there
usually means the disk is full or the OS blocked the folder. The log line
`[Save] slot=… path=…` shows the exact path.

### Save loads but progress is wrong

Check the schema version:

```
[Load] slot=1 … version=4
```

If `version` is lower than `HarvestSaveSystem.CurrentVersion`, the repair pass in
`Migrate()` ran — it fixes out-of-range values but cannot invent content that no
longer exists (e.g. a removed weapon id).

### A corrupt save is not loading

The loader falls back to `<slot>.json.bak` automatically and logs
`primary save corrupt; loaded .bak`. If both fail, the file is unrecoverable;
delete it to start a new slot.

## Build

### `Linux build module is not installed`

Unity Hub → **Installs → Add modules → Linux Build Support (Mono)**.

### The build succeeds but the smoke test fails

Open `BuildLogs/SMOKE_LOG.txt` and search for `Exception`. The smoke test stops
at the first unhandled exception, so the last `SMOKE …` line before the error
tells you which system broke.

### Build log says `another Unity instance is running`

Kill lingering `Unity.exe` processes (see above) and rebuild.

### The build is enormous

Development builds include the Mono runtime and debug symbols. For a release
build, enable **Development Build = off**, switch to IL2CPP, and enable code
stripping.

## Fast Enter Playmode

### State survives between Play sessions

A static field is missing its reset. Add it to the class's
`[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]`
method. See [Architecture](architecture/overview.md#static-state-and-fast-enter-playmode).

### Only some systems reset

Check that the class owns the field. `HarvestState`, `Preferences`, `Bindings`,
`Lexicon`, `SoundDirector`, `HarvestSaveSystem`, `Stage3D`, `HarvestHud`,
`Creature`, `HexBolt`, `NavGrid` and `RoomCollision` all have reset methods;
anything new needs its own.

## Getting help

If none of the above helps, open an issue with:

- Unity version and OS;
- target platform;
- the exact error text and the surrounding log lines;
- whether `.\Tools\build.ps1 -RunSmoke` passes.

<https://github.com/camenduru/ChosenOfTheHarvestDoc/issues>
