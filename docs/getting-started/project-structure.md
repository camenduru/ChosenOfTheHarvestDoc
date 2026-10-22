# Project structure

Everything that ships is nested under one root folder, as the Asset Store
submission guidelines require.

```
Assets/
└── ChosenOfTheHarvest/
    ├── Documentation/           offline manual (Markdown)
    ├── Editor/                  editor-only tooling
    ├── Resources/               runtime-loaded art, audio and text
    ├── Scenes/                  the single bootstrap scene
    ├── Scripts/                 all gameplay code
    ├── Settings/                URP pipeline assets
    └── Third-Party Notices.txt
```

`Resources` is nested on purpose. `Resources.Load` searches **every** folder
named `Resources`, so the load paths (`Art/...`, `Audio/...`,
`Localization/...`) stay the same while the package keeps a single root folder.

## Scenes

| File | Contents |
|---|---|
| `Scenes/ChosenOfTheHarvest.unity` | One `Game` object with the `HarvestGame` component |

There are no prefabs, no additive scenes and no scene-object references. If the
scene is ever lost, **Tools → Chosen of the Harvest → Create or Repair Bootstrap
Scene** rebuilds it.

## Editor tooling

| File | Type | Purpose |
|---|---|---|
| `Editor/BuildGame.cs` | `BuildGame` | Creates the bootstrap scene, applies player settings, builds Windows/Linux/macOS |
| `Editor/ArtImporter.cs` | `ArtImporter` | Forces every PNG under `Resources/Art` to import as a single FullRect sprite with the project's pixels-per-unit |

## Scripts

### Core

| File | Type | Responsibility |
|---|---|---|
| `Core/GameBootstrap.cs` | `HarvestGame` | Entry point and game loop: modes, hub, ceremonies, building, menus, save/load, photo mode |
| `Core/GameState.cs` | `HarvestState`, `HarvestSaveData`, `CompanionRecord`, `SanctumData`, `MemorialData`, `Curse` | Single source of truth and the save schema |
| `Core/SaveSystem.cs` | `HarvestSaveSystem` | Slots, atomic writes, `.bak`, schema migration |
| `Core/QuestSystem.cs` | `SagaSystem` | Nine-stage quest chain |
| `Core/CampaignPacing.cs` | `JourneyPacing` | Act, room and time budget constants |
| `Core/Settings.cs` | `Preferences` | Volume, difficulty, UI scale, language, autosave |
| `Core/Controls.cs` | `Bindings` | Rebindable actions |
| `Core/Localization.cs` | `Lexicon` | JSON localisation with fallback |
| `Core/AudioManager.cs` | `SoundDirector` | Music/SFX, routing, procedural fallback |
| `Core/AudioArchive.cs` | `SoundBank` | The 21 audio keys |
| `Core/DayCycle.cs` | `SeasonCycle` | Time of day and colour grading |
| `Core/View3D.cs` | `Stage3D` | Camera, billboards, ground projection, sort order |
| `Core/SmokeTest.cs` | `SmokeTest` | The `-smoke` verification run |

### Gameplay

| Folder | Key types | Responsibility |
|---|---|---|
| `Buildings/` | `Sanctum`, `SanctumType` | Structures, levels, condition, production, interiors |
| `Characters/` | `PoseAnimator` | Sprite posing and animation |
| `Combat/` | `ArmamentCatalog`, `RiteCatalog`, `HeirloomCatalog`, `AttireCatalog` | Combat data |
| `Dungeon/` | `Dungeon`, `Expedition`, `Creature`, `WardenController`, `WardenArena`, `FloorPlan`, `RoomCollision`, `Projectile`, `DungeonTrap`, `PressurePlate` | Expeditions, rooms, enemies, bosses |
| `Followers/` | `CompanionAgent`, `CompanionCatalog`, `NavGrid`, `CompanionNameTag` | The follower simulation |
| `Player/` | `PlayerController` | Movement, dodge, attack, rites, bombs, potions |
| `UI/` | `HarvestHud` | Every screen, built in code |
| `World/` | `HarvestAltar`, `GatherNode` | Altar and gatherables |

## Resources

| Path | Contents | Loaded as |
|---|---|---|
| `Art/Characters/` | 11 character sprites | `Art/Characters/<name lowercase>` |
| `Art/World/` | ground, rock, tree, altar, statue | fixed paths |
| `Art/Sanctums/` | 20 structure sprites | `Art/Sanctums/<enum lowercase>` |
| `Art/Expedition/` | floor, door, candle, skull, projectile | fixed paths |
| `Art/Wardens/` | boss body parts | fixed paths |
| `Art/UI/` | omen icon | fixed path |
| `Audio/` | 21 WAV clips | `Audio/<key>` |
| `Localization/` | `en.json`, `xx.json` | `Localization/<locale>` |

## Outside `Assets/`

| Path | Ships in the package? | Purpose |
|---|---|---|
| `ProjectSettings/` | Yes | Player, quality, input and URP settings |
| `Packages/manifest.json` | Yes | Declared Unity packages |
| `Tools/build.ps1` | Optional | Local build and smoke helper |
| `BuildLogs/Screenshots/` | **No** | Store-listing screenshots from the smoke run |
| `Library/`, `Logs/`, `Build/`, `BuildLogs/`, `UserSettings/` | **No** | Generated |

!!! warning "Never ship generated folders"

    `Build/` contains executables and DLLs. The Asset Store does not accept
    executables inside a package. Exclude `Build/`, `Library/`, `Logs/`,
    `BuildLogs/`, `UserSettings/`, `*.csproj`, `*.sln` and `*.log`.

## Related

- [Architecture overview](../architecture/overview.md)
- [Systems map](../architecture/systems-map.md)
- [Replacing art](../customization/art.md)
