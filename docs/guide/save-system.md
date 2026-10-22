# Saving and loading

Chosen of the Harvest keeps three manual slots plus a dedicated autosave, all
written as pretty-printed JSON with `JsonUtility`. Every write is atomic within
the folder, every existing file is copied to `.bak` before it is replaced, and
an old single-file save is still picked up by slot 1. Schema version 4 is the
current shape; the migration pass repairs older files in place on load.

## The slot model

`HarvestSaveSystem` defines four slots, numbered 0-3:

| Slot | Constant | File | Appears in |
|---|---|---|---|
| 0 | `AutosaveSlot` | `save_auto.json` | Load menu only |
| 1 | `FirstSlot` | `save_slot_1.json` | Save and Load menus |
| 2 | — | `save_slot_2.json` | Save and Load menus |
| 3 | `SlotCount` | `save_slot_3.json` | Save and Load menus |

`SlotCount` is 3, so the loops in `ShowSaveSlots` run `1..3`, and
`Mathf.Clamp(slot, AutosaveSlot, SlotCount)` guards every public entry point.
The autosave is deliberately absent from the save menu so a manual save can
never overwrite it — but it is the first row of the load menu.

The last slot used is remembered in `PlayerPrefs` under
`choth_active_save_slot`. `ActiveSlot` clamps both the read and the write to
`0..SlotCount`, defaults to `FirstSlot`, and calls `PlayerPrefs.Save()`.

A fresh install therefore continues from slot 1, and `Save()` with no argument
writes the active slot. Only manual saves update `ActiveSlot`; the autosave
does not steal the active slot.

## Where the files live

Everything sits directly in `Application.persistentDataPath` — Unity's
per-user data folder (the `LocalLow` tree on Windows), with no subfolder added
by the template. `SlotPath(slot)` builds the name: `save_auto.json` for slot 0,
`save_slot_N.json` otherwise.

| Path | Written by | Purpose |
|---|---|---|
| `save_slot_1.json` … `save_slot_3.json` | `Save(1..3)` | manual slots |
| `save_auto.json` | `Autosave()` | last successful expedition return |
| `<slot>.json.bak` | `Save(slot)` | copy of the previous file |
| `<slot>.json.tmp` | `Save(slot)` | transient; deleted after the copy |
| `save.json` | pre-slot builds | legacy single save, read as slot 1 |

!!! note "Test roots"

    `UseTestRoot("ProofTest")` repoints `Root` at a subfolder of
    `persistentDataPath` and clears it, so a smoke run starts clean;
    `UseDefaultRoot()` restores normal behaviour.

## What gets written

`HarvestSaveData` is one flat `[Serializable]` class. `JsonUtility.ToJson`
serialises public fields only, so everything below is a field rather than a
property.

| Group | Fields | Notes |
|---|---|---|
| Progress | `day`, `campaignStage`, `questStage`, `devotion`, `pilgrimages`, `victorySeen`, `npcIntroSeen`, `hymns` | quest and act state |
| Resources | `wood`, `stone`, `grace`, `pearls`, `food`, `meals`, `herbs`, `medicine`, `potions`, `bombs` | clamped by `HarvestState.NotifyChanged()` |
| Combat | `health`, `weaponLevel`, `weaponType`, `armorId`, `charmId`, `riteType` | health is written from the live player on save |
| Creeds | `creedGrace`, `creedCombat`, `creedEconomy`, `creedMeta` | one id per track |
| Meta | `metaVitality`, `metaWrath`, `metaProvisions`, `metaCharisma` | devotion upgrades, 0-3 |
| World | `buildings` (`List<SanctumData>`) | type, position, level, rotation, dirt, condition, interior pieces |
| Companions | `companions`, `companionRecords` (`List<CompanionRecord>`) | count plus names, traits, needs, jobs, relationships |
| Collections | `attire`, `omens`, `heirlooms`, `memorials` | owned ids and grave records |
| Versioning | `saveVersion`, `socialVersion` | see [Schema 4](#schema-version-4-and-migration) |

Field initialisers supply the defaults: `day = 1`, `wood = 10`, `food = 8`,
`companions = 4`, `health = 100`, `weaponType = "sword"`, `armorId = "wool"`,
`charmId = "ember"`, `riteType = "smite"`, `socialVersion = 2`.

## What is not persisted

| Not saved | Why |
|---|---|
| `HarvestState.CurrentCurse`, `RunHealthPenalty` | runtime-only pilgrimage modifiers, re-rolled on entry |
| Player world position | `ApplyLoadedSave()` puts the player back at the altar, `Stage3D.P(0f, -3f)` |
| An expedition in progress | loading calls `ExitExpedition(false)` and rebuilds the hub |
| Camera distance, shake, hit-stop | view state, not progress |
| UI and audio options | `PlayerPrefs` under `choth_settings_` |
| Key bindings | `PlayerPrefs` under `choth_bind_` |
| Active slot | `PlayerPrefs` under `choth_active_save_slot` |
| Photo-mode PNGs | written next to the saves, outside the schema |

## Writing a save

`Save(int slot)` is the only writer. It persists the version, reconciles the
companion list, then replaces the slot in an order that leaves a usable file at
every step:

```csharp
public static void Save(int slot)
{
    slot = Mathf.Clamp(slot, AutosaveSlot, SlotCount);
    HarvestState.Data.saveVersion = CurrentVersion;
    HarvestState.EnsureCompanionRecords();

    string path = SlotPath(slot);
    string temp = path + ".tmp";
    string backup = path + ".bak";
    string json = JsonUtility.ToJson(HarvestState.Data, true);

    Directory.CreateDirectory(Root);
    File.WriteAllText(temp, json);
    if (File.Exists(path))
    {
        try { File.Copy(path, backup, true); }
        catch (Exception exception) { Debug.LogWarning("Save backup failed: " + exception.Message); }
    }
    File.Copy(temp, path, true);
    File.Delete(temp);

    if (slot != AutosaveSlot) ActiveSlot = slot;
    Debug.Log($"[Save] slot={slot} {path}");
}
```

| Step | Failure behaviour |
|---|---|
| Write `<slot>.json.tmp` | throws; the real file is untouched |
| Copy `<slot>.json` to `.bak` | warning only; the save continues |
| Copy `.tmp` over `<slot>.json` | throws on a locked file; `.bak` still holds the previous save |
| Delete `.tmp` | a leftover `.tmp` is ignored by loads and removed by `Delete(slot)` |

Before `Save` runs, `SaveGame()` and `SaveToSlotNow()` copy the live values that
do not write themselves back: player health, each companion's carried haul
(`FlushHaul()`), sanctum dirt and condition (`SyncSanctumState()`) and the
companion records (`SyncCompanionRecords()`).

## Loading and recovery

`Load(int slot)` has three fallbacks, in order: the legacy file, the `.bak`
copy, and failure.

```csharp
public static bool Load(int slot)
{
    slot = Mathf.Clamp(slot, AutosaveSlot, SlotCount);
    string path = SlotPath(slot);
    if (!File.Exists(path) && slot == FirstSlot && File.Exists(LegacyPath))
        path = LegacyPath;
    if (!File.Exists(path)) return false;

    if (!TryRead(path, out var loaded))
    {
        string backup = path + ".bak";
        if (!File.Exists(backup) || !TryRead(backup, out loaded))
        {
            Debug.LogError("[Load] Both save and backup failed: " + path);
            return false;
        }
        Debug.LogWarning("[Load] primary save corrupt; loaded .bak");
    }

    HarvestState.Data = loaded;
    Migrate();
    if (slot != AutosaveSlot) ActiveSlot = slot;
    HarvestState.NotifyChanged();
    return true;
}
```

The return value is the whole error contract — every caller toasts on `false`:

| Caller | Message on failure |
|---|---|
| `ContinueGame()` | `title.empty` ("No save yet") |
| `LoadFromSlot(slot)` | `slots.missing` ("That slot is empty") |
| `LoadGame()` (++f9++) | "No save found" |

`TryRead` rejects only unreadable or blank files: it reads the text, returns
`false` when the content is null or whitespace, parses with
`JsonUtility.FromJson<HarvestSaveData>` and returns whether the result is
non-null. Any exception is caught and logged as a warning.

!!! warning "One generation of backup"

    `.bak` holds the file as it was before the most recent successful write —
    not a history. Two bad saves in a row leave you with a backup of the first
    bad state. If a player reports a corrupt slot, stop playing that slot, copy
    `save_slot_N.json.bak` over `save_slot_N.json`, then load.

`SlotSummary` feeds the menus and never throws on a broken file. It returns
`"empty"` when there is no file (with the same legacy fallback for slot 1).
Otherwise it builds `MM-dd HH:mm | Day N | Act N` from the file timestamp and
the parsed stage, clamping the act to `1..3`; if the JSON will not parse it
returns the timestamp alone.

## Schema version 4 and migration

`CurrentVersion` is 4 and `HarvestSaveData.saveVersion` defaults to 4.
`Save` stamps the current version before serialising, so a file always records
the build that wrote it. `Migrate()` then runs on every load, not just old
files:

```csharp
static void Migrate()
{
    if (HarvestState.Data.saveVersion < CurrentVersion)
    {
        if (HarvestState.Data.buildings != null)
            foreach (var building in HarvestState.Data.buildings)
            {
                if (building == null) continue;
                if (building.level < 1) building.level = 1;
                if (building.condition <= 0f) building.condition = 100f;
                if (building.interior == null) building.interior = new List<InteriorPieceData>();
            }

        HarvestState.EnsureLoadout();
        if (HarvestState.Data.companionRecords != null)
            foreach (var record in HarvestState.Data.companionRecords)
            {
                if (record == null) continue;
                if (record.inReflection < 0) record.inReflection = -1;
                record.priority = Mathf.Clamp(record.priority, 0, 2);
            }

        HarvestState.Data.saveVersion = CurrentVersion;
    }

    HarvestState.EnsureCompanionRecords();
}
```

There is no `switch (saveVersion)` with one branch per version. Migration is a
single idempotent repair pass: it fixes anything that would be invalid under
the current rules and moves the number forward. That keeps the code short, and
it means a v1 file and a v3 file take the same path.

| Repair | Applies to | Default it enforces |
|---|---|---|
| `building.level` | every `SanctumData` | at least 1 |
| `building.condition` | every `SanctumData` | 100 when non-positive |
| `building.interior` | every `SanctumData` | empty list, never null |
| `armorId`, `charmId`, `riteType` | `EnsureLoadout()` | a valid, owned id |
| `attire` | `EnsureLoadout()` | `wool`, `ember` |
| `inReflection` | each `CompanionRecord` | `-1` (not in the chamber) |
| `priority` | each `CompanionRecord` | clamped 0-2 |

`EnsureCompanionRecords()` handles the second version number, `socialVersion`,
and runs unconditionally at the end of `Migrate`. It upgrades the relationship
data in steps:

| From | To | Backfill |
|---|---|---|
| `< 1` | 1 | `friend` and `partner` set to `-1` on every record |
| `< 2` | 2 | `parentA`, `parentB`, `childDay` set to `-1` |

It then trims or grows `companionRecords` to `Data.companions`, fills blank
names and traits from `CompanionCatalog`, clamps needs and levels, and resolves
duplicate names. A save that predates companions entirely loads with a complete
court rather than a null list.

## F5, F9, and autosave

| Input | Binding | Method | Slot |
|---|---|---|---|
| ++f5++ | `Bindings.Action.Save` (default `Key.F5`) | `SaveGame()` | `ActiveSlot` |
| ++f9++ | `Bindings.Action.Load` (default `Key.F9`) | `LoadGame()` | `ActiveSlot` |
| — | `Autosave()` | `Save(AutosaveSlot)` | 0 |

Both hotkeys go through `Bindings.Pressed`, so rebinding them in
Settings → Gameplay → Rebind keys changes the save keys too. They are ignored
while `HarvestGame.InputLocked` is set or while a menu is open, because
`Update` returns before reaching them.

`Autosave()` respects the autosave preference, and the shipped game has exactly
one call site for it:

```csharp
public static void Autosave()
{
    if (!Preferences.Autosave) return;
    Save(AutosaveSlot);
}
// HarvestGame.ExitExpedition
if (loot) HarvestSaveSystem.Autosave();
```

So the autosave slot is refreshed when an expedition ends successfully — not on
day change, not on quest completion, not on quit. `Preferences.Autosave`
defaults to `true` and is toggled in Settings → More (`choth_settings_autosave`).

## Adding a field: a worked example

Say you add `sanctumMorale`, a float that starts every sanctum at 60 and must
never be 0 — a value older saves cannot contain. This is the full change set.

**1. Add the field** to `HarvestSaveData`, next to the other per-run counters:

```csharp
public float sanctumMorale = 60f;
```

**2. Bump the constant** in `HarvestSaveSystem`:

```csharp
public const int CurrentVersion = 5;
```

**3. Repair it in `Migrate()`**, inside the existing `saveVersion <
CurrentVersion` block and before the stamp:

```csharp
if (HarvestState.Data.sanctumMorale <= 0f)
    HarvestState.Data.sanctumMorale = 60f;

HarvestState.Data.saveVersion = CurrentVersion;
```

**4. If the field lives on a child record**, repair it in
`EnsureCompanionRecords()` instead and raise `socialVersion` with a guarded
block, the way `friend`/`partner` and the parent ids already do. Keep the
`socialVersion` gate — it is what stops a repair from re-running every load.

**5. Test against an old file.** Load a slot written by the previous build and
check the `[Load] slot=… version=…` log line, which reports the version after
migration.

`JsonUtility` leaves a field that is absent from the JSON at its default value,
so step 3 is not optional for any field whose `0`/`false`/`null` is not a
playable state. A field that is happy at zero — a counter, a flag — needs only
steps 1 and 2.

!!! tip "Version numbers are cheap"

    Bump `CurrentVersion` even when a new field needs no repair: the number is
    written into each file and shows up in every `[Load]` log line.

## Deleting a slot

`Delete(int slot)` removes all four possible files for that slot — `.json`,
`.bak`, `.tmp` and, when slot 1 is deleted, the legacy `save.json`. Each unlink
is guarded by `File.Exists`, so deleting an empty slot is a no-op.

The pause menu reaches it through `ShowDeleteSlots()`, which offers slots 1-3
only and warns "Deleting is permanent". There is no undo and no confirmation
step beyond that hint — the menu itself is the confirmation.

## Related

- [Quests and the campaign](quests.md) — the fields this schema carries
- [HUD and menus](hud.md) — the save, load and delete screens
- [State and save schema](../architecture/state-and-save.md) — the full field list
- [Save system API](../reference/core.md) - `HarvestSaveSystem` in the type reference
