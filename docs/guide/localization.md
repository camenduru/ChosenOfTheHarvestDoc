# Localization

`Lexicon` is a small static table that loads one JSON file from
`Resources/Localization` and answers `Get(key, fallback)` calls from it. English
ships as `en.json`, and `xx.json` is a pseudo-locale that prefixes every string
with `?XX?` so layout problems are impossible to miss. Both files hold the same
349 entries, and adding a language is a new JSON file plus one line in the
settings picker.

## The two tables

Each file is a single object with an `entries` array, which maps directly onto
the `Table` and `Entry` classes `JsonUtility` deserialises:

```json
{
  "entries": [
    {
      "key": "campaign.title",
      "value": "PILGRIMAGE MAP"
    },
    {
      "key": "quest.buildfarm",
      "value": "Plant a Rose Garden"
    }
  ]
}
```

```csharp
[Serializable]
class Entry
{
    public string key;
    public string value;
}

[Serializable]
class Table
{
    public Entry[] entries;
}
```

| File | Entries | Unique keys | Purpose |
|---|---|---|---|
| `en.json` | 349 | 348 | shipped English |
| `xx.json` | 349 | 348 | pseudo-locale for layout testing |

The count differs from the key count by one because `menu.assign.title` appears
twice — once as `ASSIGN`, once as `ASSIGN {0}`. The loader assigns in file
order, so the second value wins and `Lexicon.KeyCount` reports 348.

## Reading a string

Two methods cover every call in the project. Neither throws.

```csharp
public static string Get(string key, string fallback = null)
{
    Load();
    if (key != null && Values.TryGetValue(key, out var value)) return value;
    return fallback ?? key;
}

public static string Format(string key, string fallback, params object[] args)
{
    string template = Get(key, fallback);
    try
    {
        return string.Format(template, args);
    }
    catch (FormatException)
    {
        return template;
    }
}
```

| Situation | `Get` returns |
|---|---|
| key present in the loaded table | the table value |
| key absent, fallback supplied | the fallback |
| key absent, no fallback | the key itself |
| `key == null`, no fallback | `null` |

| Situation | `Format` returns |
|---|---|
| template has matching `{0}`… slots | the formatted string |
| template has no slots | the template, arguments ignored |
| template has a malformed slot | the raw template (the exception is caught) |

The practical rule: **always pass a fallback**. It is what the player sees if a
translator misses a key, and it doubles as the readable default in code review.
The fallback is the C# string, not the English table — a missing German entry
shows the source-language fallback, never `en.json`.

`Format` is used wherever a number or name is spliced in, for example the HUD
resource line:

```csharp
resourceText.text = Lexicon.Format("hud.resources",
    "Wood {0}     Stone {1}     Pearls {2}     Food {3}",
    data.wood, data.stone, data.pearls, data.food);
```

## Key families

Keys are dotted and grouped by screen or system, which makes a missing
translation easy to locate.

| Prefix | Entries | Examples |
|---|---|---|
| `menu.` | 36 | `menu.companions.title`, `menu.next`, `menu.assign.title` |
| `dungeon.` | 31 | floor, key and room copy |
| `shop.` | 22 | stock lines and prices |
| `campaign.` | 21 | `campaign.act1`, `campaign.cleared`, `campaign.victory.title` |
| `building.` | 20 | `building.farm` ("Rose Garden"), `building.temple` ("Grand Temple") |
| `prompt.` | 19 | contextual ++e++ prompts |
| `companion.` | 17 | actions, blessings, costs |
| `npc.` | 16 | Oracle intro plus one hint per quest stage |
| `settings.` | 15 | every row and On/Off |
| `slots.` | 14 | save-slot labels and toasts |
| `title.` | 11 | title screen and prologue |
| `quest.` | 11 | nine stage titles, `quest.title`, `quest.reward` |
| `ceremony.` | 10 | the six altar options and their results |
| `hud.` | 7 | `hud.day`, `hud.fps`, `hud.resources`, `hud.boss.unknown` |

Smaller families follow the same pattern: `trait.`, `room.`, `mood.`,
`recipe.`, `meta.`, `daily.`, `pause.`, `rebind.`, `phase.`, `creed.`,
`priority.`, `act.`, `forge.`, `curse.` and a single `photo.prompt`.

Four families are addressed **dynamically**, so adding content adds keys without
touching the loader:

```csharp
// Building.cs
return Lexicon.Get("building." + Type.ToString().ToLowerInvariant(), fallback);

// DayCycle.cs
Lexicon.Get("phase." + Phases[PhaseIndex].ToLowerInvariant(), Phases[PhaseIndex]);

// FloorPlan.cs
return Lexicon.Get("room." + type.ToString().ToLowerInvariant(), fallback);

// FollowerCatalog.cs
Lexicon.Get("trait." + trait.ToString().ToLowerInvariant(), trait.ToString());
```

A new `SanctumType.Beehive` needs `building.beehive` in every locale; if you
forget, the enum name appears in the UI instead of the pretty label.

## The pseudo-locale

`xx.json` is a full copy of `en.json` with every value prefixed by `?XX? ` and
identical keys. It exists to stress the layout without a real translation:

- Every string grows by five characters, which exposes labels that only fit at
  English length.
- Every string is visibly fake, so a hard-coded literal stands out next to the
  prefixed ones.
- Because the keys are identical, `KeyCount` should still report 348 — a lower
  number means the pseudo file drifted.

It is selected from **Settings → Gameplay → Language**, which shows
`Pseudo (layout)` when the preference is `xx` and `English` otherwise.

!!! tip "Ship with `xx` disabled, not deleted"

    Leaving `xx.json` in the project costs one TextAsset and makes it trivial
    to re-check a new screen for clipping. If you remove it, remove the picker
    branch too, or choosing the second language silently falls back to every
    fallback string in code.

## Hot switching

There is no scene reload. The locale is a string, the cache is a dictionary,
and the next `Get` after a switch reloads the table:

```csharp
public static void SetLocale(string next)
{
    if (string.IsNullOrWhiteSpace(next)) return;
    locale = next;
    loaded = false;
    Values.Clear();
}

static void Load()
{
    if (loaded) return;
    loaded = true;
    Values.Clear();

    var asset = Resources.Load<TextAsset>("Localization/" + locale);
    if (asset == null) return;
    // …JsonUtility.FromJson<Table>…
}
```

`Preferences.Apply()` calls `Lexicon.SetLocale(Language)` for you, and
`ShowPreferences` calls `SetLocale` directly when the language row is pressed so
the menu redraws immediately.

What refreshes instantly and what does not:

| Element | After a switch |
|---|---|
| HUD lines, prompts, toasts | next frame — `Refresh()` re-reads every `Lexicon.Format` call |
| The settings menu | immediate — the row rebuilds itself after the change |
| Any other open menu | keeps its old text until it is closed and reopened |
| Already-spawned labels, dialogue lines | unchanged; they were resolved once at spawn time |

Because `Load()` is lazy, a switch costs one `Resources` load on the next
`Get`, not at the moment of the click.

!!! warning "A missing file is silent"

    If `Resources/Localization/<locale>.json` does not exist or is not imported
    as a TextAsset, `Load()` returns with an empty dictionary and **no error**.
    `Lexicon.Locale` still reports the requested id, so the only symptom is a
    screen full of C# fallbacks. Check `Lexicon.KeyCount` — 0 means the file
    never loaded.

## Adding a language

**1. Copy `en.json`** to a new two-letter name, for example `de.json`, in
`Assets/ChosenOfTheHarvest/Resources/Localization/`.

**2. Translate the `value` fields only.** Keys, order and placeholder tokens
must survive untouched:

```json
{
  "key": "hud.day",
  "value": "Tag {0}"
}
```

Dropping or renumbering `{0}` does not crash anything — `Format` catches the
`FormatException` and returns the unformatted template — but the player sees the
braces.

**3. Leave the key set complete.** There is no fallback chain between locales,
so every key you omit shows the C# fallback string in the middle of translated
UI. Diff the key lists before shipping:

```powershell
$a = (Get-Content en.json -Raw | ConvertFrom-Json).entries.key
$b = (Get-Content de.json -Raw | ConvertFrom-Json).entries.key
Compare-Object $a $b
```

**4. Extend the picker.** The shipped settings page is a two-way toggle, so it
only knows `en` and `xx`:

```csharp
case 1:
    Preferences.SetLanguage(Preferences.Language == "en" ? "xx" : "en");

    Lexicon.SetLocale(Preferences.Language);
    ShowPreferences(2);
    break;
```

Replace it with a cycle over the locales you ship, for example
`en → xx → de → en`, and update the label on the same row (which is currently
the hard-coded `"Language   •   ..."` string in `ShowPreferences`).

**5. Persist and apply.** `Preferences.SetLanguage` writes
`choth_settings_language`; `Preferences.Apply()` re-pushes the locale on boot:

```csharp
public static void Apply()
{
    Load();
    // …audio, HUD, fullscreen…
    Lexicon.SetLocale(Language);
}
```

**6. Verify in play.** Load the language, open the quest log, the settings pages
and a build menu, and check that `Lexicon.KeyCount` matches the source table.
Because the HUD refreshes every frame, a missing key is visible immediately as a
code-style fallback string among translated text.

!!! note "Locale ids are filenames"

    `SetLocale("de")` loads `Resources/Localization/de.json`. An id like
    `de-DE` works only if the file is named `de-DE.json`; there is no region
    fallback from `de-DE` to `de`.

## Related

- [The Lexicon type](../reference/core.md) - the lookup API behind these keys
- [HUD and menus](hud.md) — every string that refreshes per frame
- [Replacing audio](../customization/audio.md) — the sibling `Resources` pipeline
- [Rebranding the template](../customization/rebranding.md) — renaming the game and its copy
