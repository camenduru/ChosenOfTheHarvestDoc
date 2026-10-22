# Core API reference

This page is the API surface of the `ChosenOfTheHarvest` core namespace: the
bootstrap, the state facade and save types, settings, input, text, audio, quests,
the day cycle and the 3D staging helpers. Signatures and defaults are taken from
the source; internal and private members are omitted, and "default" means the
value assigned in source.

## Types

### HarvestGame

`Core/GameBootstrap.cs` — the single entry point. Builds the world in `Awake`,
dispatches input in `Update`, owns every menu. Access it through
`HarvestGame.Instance`.

| Member | Type | Description |
|---|---|---|
| `Version` | `const string` | `"1.0.0"`; shown in the pause title |
| `Instance` | `static HarvestGame` | Set at the top of `Awake` |
| `InputLocked` | `static bool` | Global gate read by player, companions, sanctums, day cycle |
| `CeremonyType` | nested `enum` | `Hymn = 0`, `Feast = 1`, `Sacrifice = 2`, `Plenty = 3`, `Warding = 4` |
| `Cam` | `Camera` | Generated perspective camera (45° pitch, FOV 60) |
| `HarvestAltarRef` | `HarvestAltar` | The hub altar |
| `Player` | `PlayerController` | The Sovereign |
| `ExpeditionRef` | `Expedition` | Dungeon component, inactive in the hub |
| `HarvestHudRef` | `HarvestHud` | Generated HUD |
| `Audio` | `SoundDirector` | Generated audio director |
| `StatusLine` | `string` | Rewritten by `UpdateStatus` every frame |
| `InExpedition` / `InInterior` | `bool` | Mode queries |
| `IsNight` / `IsDusk` | `bool` | Delegated to `SeasonCycle` (dusk is phase 2) |
| `Companions` | `IReadOnlyList<CompanionAgent>` | Live agents, index-aligned with records |
| `Sanctums` | `IReadOnlyList<Sanctum>` | Placed buildings, index-aligned with `Data.buildings` |
| `HasTeaHouse` | `bool` | Any `SanctumType.TeaHouse` exists |
| `FoodSpot` / `HaulSpot` | `Vector3` | First kitchen / storage offset, else the altar |
| `FeastCost` | `int` | `4` with a kitchen, otherwise `8` |
| `IsPlacing` / `PhotoModeActive` | `bool` | Placement ghost / photo mode |
| `CampaignStage` | `int` | `Data.campaignStage` clamped 0–3 |
| `AverageHunger` | `float` | Mean hunger, `100` when there are no companions |
| `SpawnCompanion(int index)` | `void` | Builds the agent for an existing record |
| `RecruitCompanion()` | `bool` | Adds a record + agent; false at 30 companions |
| `CompanionDied(CompanionAgent, string cause = "starved to death")` | `void` | Records a memorial, removes the agent, opens the menu |
| `UsePotion()` | `void` | Consumes one potion to heal |
| `ConsumeFoodFor(CompanionAgent)` | `bool` | Pays food/meals for one companion |
| `DepositHaul(CompanionAgent, SanctumType, int amount)` | `void` | Credits a hauled load at the storage/altar spot |
| `Interact()` | `void` | Context action for hub, interior or dungeon |
| `EnterExpedition()` | `void` | Starts the current act, clamped 0–2 |
| `EnterExpedition(int act)` | `void` | Rolls a curse, disables the hub, calls `Expedition.Begin` |
| `ExitExpedition(bool loot)` | `void` | Ends the run, restores the hub, autosaves when `loot` |
| `WardenDefeated(int act, string bossName)` | `void` | Advances `campaignStage`, pays devotion, evaluates the saga |
| `ShowTitleScreen()` / `OpenPauseMenu()` | `void` | Title and pause menus (both pause and lock input) |
| `LoadGame()` | `void` | Loads the active slot and rebuilds the hub from it |
| `Build(SanctumType)` | `Sanctum` | Legacy instant build in a free row slot; pays wood/stone |
| `BeginPlacement(SanctumType)` / `BeginMove(Sanctum)` / `CancelPlacement()` | `void` | Ghost placement lifecycle |
| `SetPlacementCursor(Vector3)` | `void` | Moves the ghost, callable by UI or tests |
| `ConfirmPlacement()` / `PlaceAt(SanctumType, Vector3)` | `bool` | Commits the ghost / places directly (`ProofTest` uses `PlaceAt`) |
| `RepairSanctum` / `UpgradeSanctum` / `RemoveSanctum(Sanctum)` | `bool` | Condition, level (max 3), demolition with half refund |
| `EnterInterior(Sanctum)` / `ExitInterior()` | `void` | Builds/destroys the interior root |
| `ShowInteriorBuildMenu()` | `void` | Interior piece menu |
| `PlaceInteriorPiece(string type)` | `bool` | Places a wall/decor piece on the interior grid |
| `ResolveInteriorPosition(Vector3, float radius)` | `Vector3` | Collision resolve used by `PlayerController` |
| `ShowCeremonyMenu()` / `StartCeremony(CeremonyType)` | `void` | Altar menu and ceremony coroutine |
| `SharpenBlade()` | `void` | 15 pearls for `weaponLevel++` |
| `ShowCompanionRoster(int page = 0)` | `void` | Roster and per-companion actions |
| `ShowCampaignMap()` | `void` | Act map; entering the portal opens this |
| `ShowQuestLog()` / `ShowInventory()` | `void` | Saga progress and item list |
| `ShowAttireMenu(int page = 0)` | `void` | Equip owned armour/charms |
| `ShowOmenMenu(Action onDone)` | `void` | Offers up to two unowned omens; always calls `onDone` |
| `GrantOmen(string id, string name)` | `void` | Adds an omen id, toasts, special-cases `sun` |
| `ShowCatalogMenu()` | `void` | Read-only content browser |
| `ShowCatalogSection(int section, int page = 0)` | `void` | 0 weapons, 1 omens, 2 heirlooms, 3 equipment, 4 buildings, 5 enemies |
| `CatalogSummary()` | `string` | One-line counts of every catalog |
| `CycleWeapon()` / `CycleRite()` | `void` | Equips `Next()` and persists the id |
| `EnterPhotoMode()` / `ExitPhotoMode()` | `void` | Free camera; F12 captures a PNG to `Photos/` |
| `SetCameraDistance(float)` | `void` | Follow-distance target, minimum 6 |
| `Shake(float strength, float duration)` | `void` | No-op when screen shake is off or reduced motion is on |
| `HitStop(float duration)` / `HitFeedback(Vector3, float)` | `void` | Brief time freeze; feedback is shake + 35 ms stop |
| `FlashTransition()` | `void` | HUD fade |
| `Toast(string)` | `void` | Forwards to `HarvestHud.Toast` |
| `FloatingText(Vector3, string, Color)` | `void` | `TextMesh` that self-destructs after 1.2 s |
| `ToggleMute()` | `void` | Toggles audio and toasts the new state |

### HarvestState

`Core/GameState.cs` — static facade over the save object. Every bonus and cap
lives here.

| Member | Type | Description |
|---|---|---|
| `Data` | `HarvestSaveData` | The live save; replaced by `Reset` and by load |
| `CurrentCurse` / `RunHealthPenalty` | `Curse` / `int` | *Runtime-only* curse and its 25-health penalty |
| `Changed` | `event Action` | Raised at the end of `NotifyChanged` |
| `StorageLevels` | `int` | Sum of `max(1, level)` over storage buildings |
| `WoodCap` / `StoneCap` | `int` | `250 + 100 × StorageLevels` |
| `FoodCap` | `int` | `100 + 50 × StorageLevels` |
| `HerbCap` / `MealCap` | `int` | `30 + 20 × StorageLevels` |
| `MedicineCap` | `int` | `10 + 5 × StorageLevels` |
| `BombCap` | `int` | `5 + 2 × storage + omens/heirlooms` |
| `PotionCap` | `int` | `20`, `+5` with the `hermit` omen |
| `NotifyChanged()` | `void` | Clamps caps, then invokes `Changed` |
| `HasAttire(string id)` | `bool` | Id is in `Data.attire` |
| `EnsureLoadout()` | `void` | Repairs invalid/missing armour, charm and rite ids |
| `AddAttire` / `EquipAttire(string id)` | `bool` | Owns / equips; false when unknown or not owned |
| `GraceMultiplier` | `float` | Creed × omen × heirloom grace multiplier |
| `GatherBonus` | `int` | `+1` with the `industry` creed |
| `BonusDamage` | `int` | `+8` with the `zeal` creed |
| `BonusPearls` / `BonusFood` | `int` | Per-kill pearl / per-harvest food bonuses |
| `MaxHealth` | `int` | Full formula, floor 40, minus `RunHealthPenalty` |
| `WeaponDamage` | `int` | Armament base + upgrades + bonuses, rounded |
| `RiteCooldownMultiplier` / `RiteDamageMultiplier` | `float` | Product of charm/armour/omen/heirloom factors |
| `CriticalChance` / `CriticalMultiplier` | `float` | Base `0.08` / `1.6` plus bonuses |
| `AttackCooldownMultiplier` | `float` | Hourglass and wolf charm factors |
| `AttireLifesteal` | `int` | `1` with the `serpent` charm |
| `HasOmen(string id)` / `HasHeirloom(string id)` | `bool` | Collection membership |
| `HeirloomCount` | `int` | Owned heirlooms, 0 if null |
| `MoveSpeedMultiplier` / `DamageTakenMultiplier` | `float` | Movement and incoming-damage products |
| `ReflectDamage` | `int` | `4` with `briar` armour |
| `HeirloomSummary()` / `OmenSummary()` | `string` | Comma / dot separated display names |
| `CurseName` / `CreedName` | `string` | Display strings for the HUD |
| `RollCurse()` / `ClearRunState()` | `void` | Randomise the run curse / clear it |
| `CreedSummary()` | `string` | Joined creed names, or the `creed.none` string |
| `CreateCompanionRecord()` | `CompanionRecord` | Unique name, random trait, tint, age 18–35 |
| `EnsureCompanionRecords()` | `void` | Reconciles records with `Data.companions`, migrates, clamps, de-dupes names |
| `CompanionAt(int index)` | `CompanionRecord` | Null-safe indexed access; ensures records first |
| `Reset()` | `void` | Fresh `HarvestSaveData` + notify |

### HarvestSaveData

`Core/GameState.cs` — the serialised root. One row per field group; the
field-by-field table with defaults is in
[State and save data](../architecture/state-and-save.md#harvestsavedata-fields).

| Member | Type | Description |
|---|---|---|
| `saveVersion` | `int` | `4` |
| `day` | `int` | `1` |
| `wood`, `stone` | `int` | Bulk building materials (`10`, `0`) |
| `grace`, `pearls` | `int` | Ceremony and dungeon currencies |
| `food`, `meals`, `herbs`, `medicine`, `potions`, `bombs` | `int` | Consumables |
| `hymns`, `companions`, `health` | `int` | Hymns performed, roster size (`4`), saved health (`100`) |
| `campaignStage`, `questStage`, `devotion`, `pilgrimages` | `int` | Progression counters |
| `socialVersion` | `int` | `2`; relationship migration marker |
| `metaVitality`, `metaWrath`, `metaProvisions`, `metaCharisma` | `int` | Meta upgrade levels |
| `npcIntroSeen`, `victorySeen` | `bool` | One-shot story flags |
| `creedMeta`, `creedGrace`, `creedCombat`, `creedEconomy` | `string` | Chosen creed ids |
| `weaponLevel` | `int` | Sharpen count |
| `weaponType`, `armorId`, `charmId`, `riteType` | `string` | Equipped ids |
| `buildings` | `List<SanctumData>` | Placed sanctums |
| `attire`, `omens`, `heirlooms` | `List<string>` | Owned ids |
| `companionRecords` / `memorials` | `List<CompanionRecord>` / `List<MemorialData>` | Roster and graves |

### Preferences

`Core/Settings.cs` — persisted options in `PlayerPrefs` under `choth_settings_`.
Every setter persists immediately.

| Member | Type | Default | Description |
|---|---|---|---|
| `MasterVolume` | `float` | `0.85` | Applied to both audio sources |
| `MusicVolume` / `SfxVolume` | `float` | `0.65` / `0.85` | Music and SFX buses |
| `ScreenShake` | `bool` | `true` | Gate for `HarvestGame.Shake` |
| `ReducedMotion` | `bool` | `false` | Disables shake and hit stop |
| `Fullscreen` | `bool` | `false` | Windowed vs `FullScreenWindow` |
| `FpsCounter` / `HighContrast` | `bool` | `false` | HUD FPS label / palette switch |
| `Autosave` | `bool` | `true` | Gate for `HarvestSaveSystem.Autosave` |
| `UiScale` | `float` | `1.0` | Clamped 0.8–1.3 |
| `Language` | `string` | `"en"` | Locale file name |
| `Difficulty` | `int` | `1` | 0 easy, 1 normal, 2 hard |
| `IncomingDamageMultiplier` | `float` | — | 0.7 / 1.0 / 1.3 by difficulty |
| `CreatureHealthMultiplier` | `float` | — | 0.85 / 1.0 / 1.2 by difficulty |
| `DifficultyName` | `string` | — | `"Easy"`, `"Normal"`, `"Hard"` |
| `Load()` / `Save()` | `void` | — | Read once (`loaded` latch) / write all keys and flush |
| `SetMaster`, `SetMusic`, `SetSfx` | `void` | — | Clamp01 + persist |
| `SetScreenShake` … `SetAutosave` | `void` | — | Bool flags + persist |
| `SetUiScale`, `SetLanguage`, `SetDifficulty` | `void` | — | Clamped/validated + persist |
| `Apply()` | `void` | — | Pushes volumes, UI scale, FPS, contrast, fullscreen and locale |

### Bindings

`Core/Controls.cs` — rebindable keyboard actions. Movement, digits and Escape are
fixed and not part of this enum.

| Member | Type | Description |
|---|---|---|
| `Action` | nested `enum` | `Attack, Dodge, Interact, Build, Companions, Inventory, Map, Quest, Mute, Save, Load, Potion, Rite, Bomb` |
| `Load()` | `void` | Reads `choth_bind_<n>`, falls back to defaults once |
| `GetKey(Action)` | `UnityEngine.InputSystem.Key` | Current key |
| `GetLabel(Action)` | `string` | `Key.ToString()` |
| `SetKey(Action, Key)` | `void` | Assigns and persists |
| `ResetDefaults()` | `void` | Restores all 14 defaults and persists |
| `Pressed(Action)` / `Held(Action)` | `bool` | `wasPressedThisFrame` / `isPressed`, null-safe |

Defaults, in enum order: `Space`, `LeftShift`, `E`, `B`, `F`, `I`, `C`, `J`,
`M`, `F5`, `F9`, `H`, `X`, `G`.

### Lexicon

`Core/Localization.cs` — JSON localisation. A table is
`{ "entries": [ { "key": "...", "value": "..." } ] }` at
`Resources/Localization/<locale>.json`.

| Member | Type | Description |
|---|---|---|
| `Locale` | `string` | Active locale, default `"en"` |
| `Get(string key, string fallback = null)` | `string` | Table value, else fallback, else the key |
| `Format(string key, string fallback, params object[] args)` | `string` | `string.Format`; returns the template on format errors |
| `KeyCount` | `int` | Loaded key count |
| `SetLocale(string next)` | `void` | Switches locale and clears the cache; empty input ignored |

### SoundDirector

`Core/AudioManager.cs` — `AudioSource`-only audio with generated fallback clips.

| Member | Type | Description |
|---|---|---|
| `Instance` | `static SoundDirector` | Set in `Awake` |
| `Create(Transform parent)` | `static SoundDirector` | Creates the `Audio` GameObject and component |
| `Muted` | `bool` | Mirrors the `choth_audio_muted` pref |
| `CurrentMusic` | `string` | Key of the looping track |
| `ClipCount` | `int` | Registered clips |
| `UsedClipCount` | `int` | Keys the `SoundBank` expects to load |
| `PlaceholderClipCount` | `int` | How many of those actually loaded from `Resources/Audio` |
| `MissingPlaceholderCount` | `int` | Keys with no resource file |
| `Play(string name, float volume = 1f, float pitch = 1f)` | `void` | One-shot; pitch clamped 0.55–1.9, volume 0–1 |
| `Sfx(string name, float volume = 1f, float pitch = 1f)` | `static void` | Null-safe `Instance.Play` |
| `PlayMusic(string key)` | `void` | Loops a clip; no-op if it is already playing |
| `ToggleMute()` / `SetMuted(bool)` | `bool` / `void` | Flips (returns new state) / sets; both persist |
| `SetVolumes(float master, float music, float sfx)` | `void` | Clamp01 each, re-applies |

### HarvestSaveSystem

`Core/SaveSystem.cs` — slot files, atomic write, `.bak` recovery, migration.

| Member | Type | Description |
|---|---|---|
| `CurrentVersion` | `const int` | `4` |
| `AutosaveSlot` / `FirstSlot` / `SlotCount` | `const int` | `0` / `1` / `3` |
| `ActiveSlot` | `int` | PlayerPrefs-backed, clamped 0–3 |
| `Save()` / `Save(int slot)` | `void` | Writes `.tmp`, copies to `.bak`, replaces the slot |
| `Load()` / `Load(int slot)` | `bool` | Primary, then `.bak`, then `Migrate` |
| `Exists` / `SlotExists(int slot)` | `bool` | Active slot / any slot, including legacy `save.json` for slot 1 |
| `IsAutosave(int slot)` | `bool` | `slot == 0` |
| `Delete(int slot)` | `void` | Removes the slot, `.bak`, `.tmp` and legacy file |
| `SlotSummary(int slot)` | `string` | `"MM-dd HH:mm \| Day N \| Act N"` or `"empty"` |
| `Autosave()` | `void` | `Save(0)` when `Preferences.Autosave` |
| `UseTestRoot(string name)` / `UseDefaultRoot()` | `void` | Redirect to a scratch folder / restore `persistentDataPath` |

!!! warning "`Load` mutates global state"
    A successful `Load` assigns `HarvestState.Data` and calls `NotifyChanged`
    before returning `true`; `HarvestGame.LoadGame` then rebuilds the world. Do
    not call `Load` from inside a system that still holds old record references.

### SagaSystem

`Core/QuestSystem.cs` — the eight-step, state-driven quest chain.

| Member | Type | Description |
|---|---|---|
| `FinalStage` | `const int` | `8`; stage 8 is `quest.done` |
| `Stage` | `int` | `Data.questStage` clamped 0–8 |
| `Complete` | `bool` | `Stage >= FinalStage` |
| `CurrentTitle` | `string` | Localised title for the current step |
| `CurrentLine` | `string` | `"▶ " + CurrentTitle` |
| `CompletedLine` | `string` | Bulleted list of completed steps |
| `ProgressLine` | `string` | Campaign act, devotion and `JourneyPacing.ShortSummary` |
| `Evaluate(HarvestGame boot)` | `void` | Advances every satisfied step, pays devotion, toasts, plays UI sfx |

The reward is `+2` devotion, or `+4` once the new stage is 5 or higher. The
chain checks, in order: farm built, hymn sung, act 1 cleared, act 2 cleared, act
3 cleared, temple built, 8 companions, 30 devotion.

### SeasonCycle

`Core/DayCycle.cs` — four phases, 60 real seconds each.

| Member | Type | Description |
|---|---|---|
| `PhaseSeconds` | `const float` | `60` |
| `Phases` | `static string[]` | `Dawn`, `Day`, `Dusk`, `Night` |
| `PhaseIndex` | `int` | 0–3 |
| `PhaseName` | `string` | Localised `phase.<name>`, falls back to English |
| `IsNight` | `bool` | Phase 3 |
| `PhaseChanged` | `event Action<int>` | Fired on reset and on every advance |
| `ResetToDawn()` | `void` | Timer 0, phase 0, repaint, raise the event |

`Update` returns immediately while `HarvestGame.InputLocked`, so menus freeze the
clock. Advancing into Dawn increments `Data.day` and notifies.

### Stage3D

`Core/View3D.cs` — the presentation layer that keeps gameplay on a flat plane
while rendering it as a 3/4 perspective scene.

| Member | Type | Description |
|---|---|---|
| `CameraPitch` / `CameraDistance` / `CameraFov` | `const float` | `45`, `10`, `60` |
| `BillboardRotation` | `static Quaternion` | `Euler(45, 0, 0)` |
| `FlatRotation` | `static Quaternion` | `Euler(-90, 0, 0)` |
| `P(float x, float depth)` / `P(Vector3 gameplay)` | `Vector3` | Gameplay (x, depth) → world `(x, 0, depth)` |
| `Ground(Vector3 world)` | `Vector3` | Flattens to `y = 0` |
| `Above(Vector3 ground, float height)` | `Vector3` | Offsets along screen-up (0.707 per axis) |
| `SortOrder(Vector3 world, int bias = 0)` | `int` | `2000 - round(z × 10) + bias` |
| `GlowSprite` | `Sprite` | Lazily generated radial alpha sprite |
| `SpriteMaterial` | `Material` | URP 2D unlit sprite material, `Sprites/Default` fallback |
| `AttachBillboard(GameObject root, Sprite, Color, int order, float xOffset = 0)` | `SpriteRenderer` | Adds a camera-facing sprite to an existing root |
| `CreateBillboard(Transform parent, string name, Sprite, Vector3 ground, float height, Color, int order, float xOffset = 0, float maxWidth = 0)` | `GameObject` | Root + billboard, scaled to a target height |
| `CreateFlat(...)` | `SpriteRenderer` | Ground-plane sprite for floors, glows and zones |
| `HorizontalDistance(Vector3 a, Vector3 b)` | `float` | XZ distance |
| `CreateShadow(Transform worldParent, Transform follow, float radius)` | `void` | Adds a `GroundShadow` blob |

```csharp
var go = Stage3D.CreateBillboard(hubRoot.transform, "Shrine Sanctum", sprite,
    Stage3D.P(3f, -2f), 2.6f, Color.white, Stage3D.SortOrder(Stage3D.P(3f, -2f)));
Stage3D.CreateShadow(hubRoot.transform, go.transform, 1.2f);
```

## Related

- [Bootstrap and game loop](../architecture/bootstrap.md)
- [State and save data](../architecture/state-and-save.md)
- [Catalogs API reference](catalogs.md)
- [Controls](../getting-started/controls.md)
