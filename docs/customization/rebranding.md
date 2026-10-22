# Rebranding the template

Chosen of the Harvest keeps its identity in four places: Unity's player settings,
one C# namespace, the localisation table, and the persistence keys that name saves
and preferences. No prefab, material or scene asset carries the name, so rebranding
is a short mechanical pass rather than an asset hunt. This page is the checklist,
followed by the order that avoids breaking builds and saves.

## The rebranding map

Every identity string a player, reviewer or build machine can actually see.

| What it affects | Where it lives | Shipped value |
| --- | --- | --- |
| Window title and save folder | `ProjectSettings/ProjectSettings.asset` → `productName` | `ChosenOfTheHarvest` |
| Publisher / company | same file → `companyName`, plus `BuildGame.ConfigureProduct()` | `ChosenOfTheHarvest` |
| Bundle identifier | same file → `applicationIdentifier: Standalone` | `com.chosenoftheharvest.template` |
| Version string | `bundleVersion`, `HarvestGame.Version`, `BuildGame.Version` | `1.0.0` |
| Runtime namespace | 37 files with `namespace ChosenOfTheHarvest` | `ChosenOfTheHarvest` |
| Editor namespace | 2 files with `namespace ChosenOfTheHarvest.EditorTools` | `ChosenOfTheHarvest.EditorTools` |
| Editor menu | `[MenuItem("Chosen of the Harvest/…")]` in `BuildGame.cs` | `Chosen of the Harvest` |
| Title, subtitle, prologue | `Resources/Localization/en.json` → `title.*` | `CHOSEN OF THE HARVEST` |
| Lore noun | `en.json` → 9 values containing `Harvesthold` | `Harvesthold` |
| Pseudo-locale | `Resources/Localization/xx.json` | same strings with `?XX?` |
| Preference keys | `choth_settings_`, `choth_bind_`, `choth_audio_muted`, `choth_active_save_slot` | `choth_` prefix |
| Save files | `save_auto.json`, `save_slot_1..3.json`, `save.json`, `Photos/` | see `HarvestSaveSystem` |
| Window size | `HarvestGame.Awake()` | `1600 × 900`, windowed |

!!! note "Display names are already data"

    Sanctums, traits, quests and menu labels are looked up through `Lexicon` and
    fall back to English literals in code. Changing a *display* name is an `en.json`
    edit; changing a *type* name is a code change and a save-compatibility change.
    Keep the two separate.

## Step 1 — Set the player identity

`BuildGame.ConfigureProduct()` rewrites `companyName`, `productName` and
`bundleVersion` on every build, so editing `ProjectSettings.asset` alone is not
durable if you build through the template menus.

1. Open `ProjectSettings/ProjectSettings.asset` and edit the identity block.
2. Edit the same values in `Assets/ChosenOfTheHarvest/Editor/BuildGame.cs`.
3. Leave `applicationIdentifier` alone for now — the build helper never touches it.

```yaml
# ProjectSettings/ProjectSettings.asset
companyName: YourStudio
productName: YourGame
bundleVersion: 0.1.0
applicationIdentifier:
  Standalone: com.yourstudio.yourgame
```

```csharp
// Assets/ChosenOfTheHarvest/Editor/BuildGame.cs
public const string ProductName = "YourGame";
public const string Version = "0.1.0";

static void ConfigureProduct()
{
    PlayerSettings.companyName = "YourStudio";
    PlayerSettings.productName = ProductName;
    PlayerSettings.bundleVersion = Version;
    PlayerSettings.SetApplicationIdentifier(
        BuildTargetGroup.Standalone, "com.yourstudio.yourgame");
    // ... existing backend / window settings
}
```

`productName` is also the folder name under `%USERPROFILE%\AppData\LocalLow`, so a
space is legal but annoying in bug reports. Prefer no spaces.

!!! warning "The window size is set twice"

    `PlayerSettings.defaultScreenWidth/Height` (1280 × 720) only affects the
    standalone launch. `HarvestGame.Awake()` then calls
    `Screen.SetResolution(1600, 900, FullScreenMode.Windowed)`, which wins while
    the game runs. Change both or the player will resize itself on start.

## Step 2 — Rename the namespace

The runtime namespace is one identifier; the editor namespace is a child of it.
Rename both with your IDE's *Rename namespace* refactor, not with a text replace.

```csharp
// before
namespace ChosenOfTheHarvest { /* 37 runtime files */ }
namespace ChosenOfTheHarvest.EditorTools { /* ArtImporter.cs, BuildGame.cs */ }

// after
namespace YourStudio.YourGame { /* … */ }
namespace YourStudio.YourGame.EditorTools { /* … */ }
```

The child namespace keeps working because C# resolves parent namespaces
automatically. Any `using ChosenOfTheHarvest;` must be renamed too — the IDE
refactor does that for you.

!!! danger "A blanket text replace breaks the scene path"

    `BuildGame.ScenePath` is the literal
    `"Assets/ChosenOfTheHarvest/Scenes/ChosenOfTheHarvest.unity"`. A naive
    find-and-replace of `ChosenOfTheHarvest` rewrites that folder path and the
    editor can no longer find the bootstrap scene. Either leave the folder name
    alone (recommended) or move the folder **and** update `ScenePath`,
    `ProjectSettings/EditorBuildSettings.asset` and every `.meta` alongside it.

The CLI entry point embeds the namespace, so update any build script that calls it:

```powershell
# before: -executeMethod ChosenOfTheHarvest.EditorTools.BuildGame.BuildPlayer
$unity = "C:\Program Files\Unity\Hub\Editor\6000.6.6f1\Editor\Unity.exe"

& $unity -batchmode -quit -projectPath (Get-Location) `
  -executeMethod YourStudio.YourGame.EditorTools.BuildGame.BuildPlayer `
  -logFile build.log
```

If the string and the namespace disagree, Unity exits with
`executeMethod class not found` before compiling a single player.

## Step 3 — Retitle the title screen

`HarvestGame` asks `Lexicon` for `title.name` and only uses the uppercase literal as
a fallback. Editing the JSON is enough; no recompile is needed after a domain reload.

```json
{
  "key": "title.name",
  "value": "YOUR GAME"
},
{
  "key": "title.subtitle",
  "value": "A HAND-BUILT SAGA"
},
{
  "key": "title.prologue.2",
  "value": "Gather a harvest. Grow your sanctuary. Break the three wardens."
}
```

1. Edit the eleven `title.*` entries in `en.json`.
2. Mirror them in `xx.json` (keep the `?XX? ` prefix) or the pseudo-locale toggle shows the old brand.
3. Search `en.json` for `Harvesthold` and rewrite the nine values that mention it.
4. Reload the scene and check the title, the pause header and the prologue dialogue.

!!! warning "Duplicate key in the shipped table"

    `en.json` contains two entries for `menu.assign.title`. `Lexicon` keeps the
    **last** one it reads, so edit or delete both. A duplicate you forget will
    quietly override your new string.

## Step 4 — Re-prefix persistence (optional)

Four constants own the whole `PlayerPrefs` surface:

```csharp
// Settings.cs
const string Prefix = "choth_settings_";
// Controls.cs
static readonly string Prefix = "choth_bind_";
// AudioManager.cs
const string MutePref = "choth_audio_muted";
// SaveSystem.cs
const string ActiveSlotPref = "choth_active_save_slot";
```

Renaming them gives your players a clean slate: old preferences and the remembered
save slot are simply ignored. Existing save **files** are unaffected, because their
names are separate literals (`save_auto.json`, `save_slot_{n}.json`).

!!! tip "Rebrand, then migrate once"

    If you want a new prefix without losing settings, read the old key, write the
    new one, then delete the old one on first launch. Doing it in `Preferences.Load()`
    and `Bindings.Load()` is enough — both already run before anything reads a value.

## Step 5 — Decide what *not* to rename

| Don't rename | Why |
| --- | --- |
| `HarvestSaveData` field names | `JsonUtility` maps JSON by field name; a rename silently zeroes that field |
| `SanctumType` members | Saves store `type.ToString()`; a rename turns a Farm into a fallback shrine |
| `ArmamentDef.Id`, rite ids, omen ids | Stored in `weaponType`, `riteType`, `omens[]`; unknown ids fall back or vanish |
| `Resources/Art`, `Resources/Audio` | Hard-coded in `Resources.Load` calls throughout |
| The `choth_` prefixes mid-project | Orphans every tester's bindings, volume and active slot |

Renaming the *classes* (`HarvestGame`, `HarvestHud`, `HarvestState`) is safe for
saves — the type name is not serialised — but it touches hundreds of call sites for
no player-visible gain. Rebrand the strings, not the vocabulary.

## Step 6 — Verify the rebrand

1. Press Play: title, subtitle and prologue show the new name.
2. Open Preferences → confirm the header and any `choth_` migration behaved.
3. Build with `BuildGame.BuildPlayer` and check the window title and `Build/` output name.
4. Launch the player and confirm the save folder is
   `%USERPROFILE%\AppData\LocalLow\YourStudio\YourGame`.
5. Save, quit, relaunch, continue — the slot summary should still list the day and act.
6. Run `rg -n "Chosen of the Harvest|ChosenOfTheHarvest" Assets ProjectSettings`
   and confirm every remaining hit is intentional (folder names, class names).

## Related

- [Project structure](../getting-started/project-structure.md) — where these files sit.
- [Building players](../publishing/building.md) — the build methods used above.
- [Localization](../guide/localization.md) — the `Lexicon` pipeline in depth.
- [State & save schema](../architecture/state-and-save.md) — what must not be renamed.
