# HUD and menus

Every pixel of the interface is built at runtime with uGUI: `HarvestHud` owns
the on-screen layers, and `HarvestGame` builds each screen by filling one
reusable six-row menu. There are no prefabs, no canvas assets and no scene
wiring — which is why a menu is added by writing a method rather than by
dragging a panel.

## What is on screen

`HarvestHud.Build()` creates a `Canvas` (`ScreenSpaceOverlay`, sorting order
100) with a `CanvasScaler` set to `ScaleWithScreenSize`, reference 1920×1080,
match 0.5. All text draws through `UiFont` — `LegacyRuntime.ttf`, with a dynamic
OS font as the fallback — plus a hard `Shadow` offset of `(2, -2)`.

| Element | Built by | Placement | Notes |
|---|---|---|---|
| Day and phase | `Build()` | top-left, 30 pt | `Day 12   •   Dusk` |
| Health bar | `BuildHealthBar()` | 300×20 at `(28, -58)` | red, turns orange below 35% |
| Warden bar | `BuildWardenBar()` | 560×52, top centre | hidden until `ShowWarden` |
| Minimap | `BuildMinimapShell()` | 330×170, top-right | hidden outside expeditions |
| Resources | `Build()` | 24 pt at `(28, -92)` | wood, stone, pearls, food |
| Companions | `Build()` | 24 pt at `(28, -128)` | grace, count, omens, creed |
| Items | `Build()` | 20 pt at `(28, -164)` | meals, herbs, medicine, potions, rite |
| Status line | `Build()` | 20 pt at `(28, -198)` | `HarvestGame.StatusLine` |
| Prompt | `Build()` | 26 pt, bottom `y = 92` | contextual ++e++ hint |
| Toast | `Build()` | 28 pt, bottom `y = 150` | gold, 3.2 s default |
| FPS | `Build()` | 20 pt, top-right | off by default |
| Menu | `BuildMenu()` | full-screen overlay | six rows, title and hint |
| Fade | `BuildFade()` | full-screen black | 0.35 s flash |

The status line is the hub/expedition switch. In the hub it reports creed, hymn
count, act, devotion and the current quest title; inside an expedition it
reports biome, floor, room type, keys, husk count and the run curse:

```csharp
StatusLine = mode == Mode.Expedition
    ? $"DUNGEON   •   {ExpeditionRef.BiomeName}   •   floor {ExpeditionRef.Floor}/{ExpeditionRef.FloorCount}   " +
      $"•   {ExpeditionRef.RoomTypeLabel}   •   keys {ExpeditionRef.Keys}   " +
      $"•   {ExpeditionRef.CreatureCount} husks   •   {HarvestState.CurseName}"
    : $"{HarvestState.CreedSummary()}   •   hymn {HarvestState.Data.hymns}   •   act {Mathf.Min(CampaignStage + 1, 3)}/3   " +
      $"•   devotion {HarvestState.Data.devotion}   •   {SagaSystem.CurrentTitle}";
```

`Refresh()` runs every frame and re-reads the model, so a change made anywhere
in the game shows up without an explicit update call. That includes localization:
the HUD is the fastest way to see a language switch take effect.

### The minimap

`SetMinimap(rooms, current)` rebuilds the panel from the floor plan each room.
Connectors are drawn first as 2 px lines, then 22×22 nodes on a 56×48 px grid.
Visited rooms show a type letter and colour (`E` entrance, `C` combat,
`L` elite, `T` treasure, `S` shop, `K` curse, `P` puzzle, `B` Warden);
unvisited rooms show `?` on a dark tile. The current room gets a 30×30 gold
outline behind its node.

### The Warden bar

`ShowWarden(Creature enemy)` shows the bar and `HideWarden()` clears it.
`Refresh()` also hides the bar when the creature is dead or inactive, and hides
the whole minimap when `HarvestGame.InExpedition` is false. The name comes from
`ExpeditionRef.WardenName`, uppercased, with `hud.boss.unknown` as the fallback.

## The title screen

`HarvestGame.Awake()` ends with `OpenTitleScreen()` unless the build was launched
with `-smoke`. The title is just a menu with gameplay hidden:

```csharp
bool hasSave = HarvestSaveSystem.SlotExists(HarvestSaveSystem.ActiveSlot);
string continueLine = Lexicon.Get("title.continue", "Continue") + "   •   " +
                      (hasSave
                          ? HarvestSaveSystem.SlotSummary(HarvestSaveSystem.ActiveSlot)
                          : Lexicon.Get("title.empty", "No save yet"));
```

| Row | Action |
|---|---|
| New game | `StartNewGame()` — resets `HarvestState`, respawns the court, shows the prologue |
| Continue | `ContinueGame()` — loads `ActiveSlot`, or toasts "No save yet" and reopens |
| Load game | `ShowSaveSlots(true)` |
| Preferences | `ShowPreferences(0)` |
| Quit | `Application.Quit()` |

The title name and subtitle are `title.name` and `title.subtitle`. ++esc++ on
the title reopens it (the `choice < 0` branch), so there is no state to fall out
of.

## The pause menu

++esc++ or the gamepad Start button calls `OpenPause()`, which refuses while
already paused, in photo mode, during a ceremony, or while placing a building.
It stops time, locks input and shows six rows:

| Row | Action |
|---|---|
| Resume | `ResumeGame()` |
| Save game | `ShowSaveSlots(false)` |
| Load game | `ShowSaveSlots(true)` |
| Preferences | `ShowPreferences()` |
| Photo mode | `EnterPhotoMode()` |
| Quit to title | `OpenTitleScreen()` (the `pause.quit` label; the code's fallback reads "Quit to desktop") |

The title is `PAUSED   •   v1.0.0`, using `HarvestGame.Version`, and the hint is
`Press 1-6   •   Esc resumes`. Quit returns to the title screen rather than
closing the application.

## Settings

`ShowPreferences(int page = 0)` is three pages of ordinary menu rows. Audio and
motion live on page 0, display and saving on page 1, gameplay and controls on
page 2.

| Page | Title | Rows |
|---|---|---|
| 0 | `SETTINGS` | master, music, SFX, screen shake, reduced motion, More options |
| 1 | `SETTINGS   •   MORE` | UI scale, fullscreen, FPS counter, autosave, Gameplay and controls, Back |
| 2 | `SETTINGS   •   GAMEPLAY` | difficulty, language, high contrast, rebind keys, Back |

Page 0 and page 1 titles are localized; page 2's title is a hard-coded English
string (`"SETTINGS   •   GAMEPLAY"`), as are the difficulty, language and
high-contrast row labels.

Rows toggle or cycle in place and rebuild the same page, so a click never closes
the menu:

| Row | Control |
|---|---|
| Volumes | `CycleVolume` — 0% to 100% in 10% steps |
| UI scale | `CycleScale` — 0.8 to 1.3 in 0.1 steps |
| Difficulty | `(Difficulty + 1) % 3` — Easy, Normal, Hard |
| Language | `en` ↔ `xx` |
| Everything else | boolean toggle |

Every change calls `Preferences.Save()` (inside the setter) and usually
`Preferences.Apply()`, which pushes volumes to `SoundDirector`, scale/FPS/contrast
to the HUD, fullscreen mode to `Screen`, and the locale to `Lexicon`.

### Rebinding

`ShowRebindMenu(page)` is three pages over the fourteen keyboard actions:
attack, dodge, interact, rite, bomb; then potion, build, companions, inventory,
map; then quest log, mute, quick save, quick load. Choosing a row calls
`BeginRebind`, which hides the menu and sets the prompt to
`Press a key   •   Esc cancels`. `HandleRebind` captures the next key press,
writes it through `Bindings.SetKey`, and returns to the same page.

## Toasts, prompts and dialogue

A toast is one `Text` with a countdown:

```csharp
public void Toast(string text, float seconds = 3.2f)
{
    toastText.text = text;
    toastTimer = seconds;
}
```

The timer is decremented with `Time.deltaTime` in `Update`, so toasts freeze
while the game is paused and resume on unpause. `HarvestGame.Toast(string)`
forwards to this method, which is why every system can toast without holding a
HUD reference.

Prompts are simpler: `SetPrompt(text)` replaces the line at the bottom of the
screen, and `UpdatePrompt()` in `HarvestGame` decides what to show based on what
the player is standing next to.

Dialogue is a menu, not a panel. `ShowDialogue(speaker, lines, onDone)` shows one
line at a time with a Continue row, and swaps the second row for `npc.leave` on
the last line. Closing the menu with ++esc++ also fires `onDone`, which is how
the prologue unpauses the game.

## Photo mode

`EnterPhotoMode()` hides the HUD and every companion name tag, stops time and
locks input. The camera detaches from the player and pans over the scene:

| Input | Effect |
|---|---|
| ++w++ ++a++ ++s++ ++d++ or arrows | pan the focus point at 6 units/s, clamped to the world |
| Mouse scroll | zoom between 4 and 16 units |
| ++f12++ | capture |
| ++esc++ | exit to the pause menu |

`CapturePhoto()` waits for `WaitForEndOfFrame`, reads the screen into a
`Texture2D`, and writes a PNG:

```csharp
string directory = Path.Combine(Application.persistentDataPath, "Photos");
Directory.CreateDirectory(directory);
string path = Path.Combine(directory, "photo_" + System.DateTime.Now.ToString("yyyyMMdd_HHmmss") + ".png");
File.WriteAllBytes(path, texture.EncodeToPNG());
```

The file name is timestamped to the second, so two captures in the same second
overwrite each other. `ExitPhotoMode()` restores the HUD and name tags and
reopens the pause menu.

## The full-screen menus

All of these are `ShowMenu` calls, so they share navigation, the dark overlay
and the numbered rows. Keys are the defaults from `Bindings` and the fixed
`Update` branches.

| Key | Screen | Method | Availability |
|---|---|---|---|
| ++i++ | Inventory and equipment | `ShowInventory()` | hub |
| ++k++ | Catalog (weapons, omens, heirlooms, equipment, buildings, enemies) | `ShowCatalogMenu()` | hub |
| ++f++ | Companion roster | `ShowCompanionRoster()` | hub |
| ++c++ | Pilgrimage map | `ShowCampaignMap()` | hub |
| ++j++ | Quest log | `ShowQuestLog()` | anywhere |
| ++b++ | Build menu | `ShowBuildMenu()` / `ShowInteriorBuildMenu()` | hub or interior |
| ++r++ | Sanctum menu for the nearest building | `ShowSanctumMenu(building)` | hub, within 2.4 units |
| ++e++ | Interact — altar, portal, NPC, node | `Interact()` | context |

Long lists page rather than scroll. The catalog and the roster show five rows
plus a Next/Previous row; when everything fits on one page, that sixth row is
simply unused.

!!! warning "Six rows is the hard limit"

    `HarvestHud` allocates exactly six option texts and six icon slots. A
    `ShowMenu` call with more than six strings silently drops the extras, and
    every screen above is designed around that cap (five rows plus paging).
    Raise the array size in `BuildMenu` before adding a seventh row.

## Keyboard and gamepad navigation

`HarvestHud.HandleMenuInput()` runs only while a menu is open:

| Input | Result |
|---|---|
| ++1++ … ++6++ | choose that row |
| ++esc++ | close with `-1` (cancel) |
| Gamepad d-pad up/down | move the highlight |
| Gamepad left stick up/down | move the highlight |
| Gamepad South (A) | confirm the highlighted row |
| Gamepad East (B) | cancel |
| Gamepad Start | open the pause menu (from `HarvestGame.Update`) |

The highlight is a colour swap to `(1, 0.86, 0.4)` on the selected row, and
`MoveMenuSelection` skips rows that are inactive, so paged menus cannot land on
an empty slot.

Every visible row is prefixed with its number (`1)   Resume`), which is why the
hints read `Press 1-6`. `ShowMenu` also sets `HarvestGame.InputLocked = true`
and `HideMenu`/`Close` clears it, so world input is off for as long as a menu is
up.

!!! note "A keyboard is required to drive menus"

    `HandleMenuInput` starts with `var kb = Keyboard.current; if (kb == null) return;`
    before it reads the gamepad. On a device with a gamepad and no keyboard
    recognised by the Input System, menus render but cannot be navigated. Add a
    null-safe path for `kb` if you ship to such a device.

## Accessibility and display

Four options change the HUD rather than the game:

| Option | Applied by | Effect |
|---|---|---|
| UI scale | `SetUiScale` | `CanvasScaler.scaleFactor`, clamped 0.8-1.3 |
| High contrast | `SetHighContrast` | gold prompt, near-white status, pale green items, near-black menu |
| FPS counter | `SetFpsVisible` | shows the smoothed top-right counter |
| Reduced motion | `Preferences.ReducedMotion` | `Shake` and `HitStop` return immediately |

High contrast deliberately touches only the prompt, status, item and menu
background — the day line, resource line, health bar and boss bar keep their
colours. If you extend it, `SetHighContrast` is the single place to do it.

```csharp
public void SetUiScale(float scale)
{
    if (canvasScaler != null) canvasScaler.scaleFactor = Mathf.Clamp(scale, 0.8f, 1.3f);
}
```

The FPS readout uses unscaled time and an exponential smooth, so it keeps
updating while the game is paused but only if it is visible:

```csharp
float instant = Time.unscaledDeltaTime > 0f ? 1f / Time.unscaledDeltaTime : 60f;
fpsSmooth = Mathf.Lerp(fpsSmooth, instant, 0.08f);
fpsText.text = Mathf.RoundToInt(fpsSmooth) + " " + Lexicon.Get("hud.fps", "FPS");
```

## Where the code lives

| Concern | File | Members |
|---|---|---|
| Canvas, bars, text, menu shell | `Scripts/UI/Hud.cs` | `HarvestHud.Build`, `BuildHealthBar`, `BuildWardenBar`, `BuildMinimapShell`, `BuildMenu`, `BuildFade` |
| Per-frame values | `Scripts/UI/Hud.cs` | `Refresh`, `SetMinimap`, `ShowWarden`, `HideWarden` |
| Menu behaviour | `Scripts/UI/Hud.cs` | `ShowMenu`, `HideMenu`, `Choose`, `HandleMenuInput`, `MoveMenuSelection` |
| Display options | `Scripts/UI/Hud.cs` | `SetUiScale`, `SetFpsVisible`, `SetHighContrast`, `SetGameplayVisible` |
| Title, pause, slots, settings | `Scripts/Core/GameBootstrap.cs` | `OpenTitleScreen`, `ShowPauseMenu`, `ShowSaveSlots`, `ShowPreferences`, `ShowRebindMenu` |
| Photo mode | `Scripts/Core/GameBootstrap.cs` | `EnterPhotoMode`, `PhotoUpdate`, `CapturePhoto`, `ExitPhotoMode` |
| Inventory, catalog, roster, map, quest log | `Scripts/Core/GameBootstrap.cs` | `ShowInventory`, `ShowCatalogMenu`, `ShowCompanionRoster`, `ShowCampaignMap`, `ShowQuestLog` |
| Prompts, dialogue, toasts | `Scripts/Core/GameBootstrap.cs` | `UpdatePrompt`, `ShowDialogue`, `Toast` |

## Related

- [Bindings and defaults](../reference/core.md) - every default binding and the rebind flow
- [Quests and the campaign](quests.md) — what the quest log and map display
- [Saving and loading](save-system.md) — the slot menu behind Save and Load
- [Rebranding the template](../customization/rebranding.md) — retitling the generated screens
