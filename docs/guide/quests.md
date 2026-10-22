# Quests and the campaign

`SagaSystem` runs a nine-stage chain that walks a new Sovereign from the first
Rose Garden to the death of the third Warden. Every stage is a question asked
about the save file rather than an event you have to remember to raise, so a
quest can never get stuck waiting for a callback that was never sent. Press
++j++ at any point to read the log.

## The nine-stage chain

The English copy below is exactly what ships in `en.json`; the condition column
is the code that decides the stage is finished. Stage numbers are the value of
`HarvestState.Data.questStage`.

| Stage | Quest key | Objective (English copy) | Completes when | Devotion |
|---|---|---|---|---|
| 0 | `quest.buildfarm` | Plant a Rose Garden | a `Sanctum` with `Type == SanctumType.Farm` exists | +2 |
| 1 | `quest.hymn` | Hold a hymn at the Harvest Altar | `HarvestState.Data.hymns > 0` | +2 |
| 2 | `quest.reflection` | Defeat the Gilded Stag in the Ashen Orchard | `campaignStage >= 1` | +2 |
| 3 | `quest.forest` | Defeat the Mire Queen in the Drowned Conservatory | `campaignStage >= 2` | +2 |
| 4 | `quest.temple` | Defeat the Obsidian Sultan in the Ember Citadel | `campaignStage >= 3` | +4 |
| 5 | `quest.sanctum` | Raise the Grand Temple | a `Sanctum` with `Type == SanctumType.Temple` exists | +4 |
| 6 | `quest.companions` | Grow the flock to eight | `HarvestState.Data.companions >= 8` | +4 |
| 7 | `quest.devotion` | Reach 30 Devotion | `HarvestState.Data.devotion >= 30` | +4 |
| 8 | `quest.done` | The Harvesthold has endured all three wardens. | terminal — `Stage >= FinalStage` | — |

Two details are easy to miss when reading the copy alone:

- The first stage says *Rose Garden* because that is the localized label of
  `SanctumType.Farm` (`building.farm`), not a separate building type.
- Stage 5 asks for a Grand Temple (`building.temple`) **after** the third
  Warden. It is a post-victory build target, not a gate on the campaign.

## How a stage advances

`HarvestGame.Update()` polls the chain once a second, and only in the hub while
input is unlocked:

```csharp
if (mode == Mode.Hub && !InputLocked)
{
    questTimer -= Time.deltaTime;
    if (questTimer <= 0f)
    {
        questTimer = 1f;
        SagaSystem.Evaluate(this);
    }
}
```

`Evaluate` is a `while` loop, so several stages can fall in a single pass:

```csharp
while (Stage < FinalStage)
{
    bool done = Stage switch
    {
        0 => HasFarm(boot),
        1 => HarvestState.Data.hymns > 0,
        2 => HarvestState.Data.campaignStage >= 1,
        3 => HarvestState.Data.campaignStage >= 2,
        4 => HarvestState.Data.campaignStage >= 3,
        5 => HasTemple(boot),
        6 => HarvestState.Data.companions >= 8,
        7 => HarvestState.Data.devotion >= 30,
        _ => true
    };

    if (!done) break;

    HarvestState.Data.questStage++;
    int reward = Stage >= 5 ? 4 : 2;
    HarvestState.Data.devotion += reward;
    HarvestState.NotifyChanged();
    boot.Toast(string.Format(Lexicon.Get("quest.reward", "Quest complete • +{0} devotion"), reward));
    SoundDirector.Sfx("ui", 0.65f, 1.15f);
}
```

Finish a Grand Temple while already holding eight companions and thirty
Devotion and the loop completes stages 5, 6 and 7 in the same second, paying
each reward separately.

`Evaluate` is also called from two event-driven places so the player does not
have to wait for the next poll:

- `HarvestGame.WardenDefeated(int act, string bossName)`, after it raises
  `campaignStage`.
- The end of `HarvestGame.CeremonyRoutine`, which is where a Hymn increments
  `HarvestState.Data.hymns`.

`Stage` is clamped to `FinalStage` on read, so a corrupt or hand-edited
`questStage` of 999 still reads as complete instead of throwing.

## Rewards and devotion

Devotion is the only currency the chain pays. The payout is decided by the
**new** stage value, because `Stage` is a property read after `questStage++`.

| Stages completed | New stage | Devotion per stage | Subtotal |
|---|---|---|---|
| 0-3 | 1-4 | 2 | 8 |
| 4-7 | 5-8 | 4 | 16 |
| **Total** | | | **24** |

The campaign pays on top of that. `WardenDefeated` awards
`Mathf.RoundToInt((8 + act * 4) * (HasHeirloom("crown") ? 1.15f : 1f))`, which
is the `boss rewards 8/12/16` line in `JourneyPacing.BalanceLine`:

| Act | Index | Base devotion | With the Crown heirloom |
|---|---|---|---|
| Ashen Orchard | 0 | 8 | 9 |
| Drowned Conservatory | 1 | 12 | 14 |
| Ember Citadel | 2 | 16 | 18 |

Devotion sinks are in `HarvestGame.ShowMetaUpgrades()`. Each track caps at
level 3 and costs its base plus 2 per level already bought:

| Track | First cost | What it buys |
|---|---|---|
| Vitality | 3 | `HarvestState.MaxHealth` gains `metaVitality * 15` |
| Wrath | 3 | `HarvestState.WeaponDamage` gains `metaWrath * 4` |
| Provisions | 2 | `+10 food` and `+5 meals` per level |
| Charisma | 2 | `+3 loyalty` to every companion per level |

!!! tip "Devotion is permanent"

    `campaignStage`, `questStage`, `devotion` and the four `meta*` levels all
    live in the save file. Clearing Act III does not reset the chain, and the
    upgrade menu reads the same `devotion` the quest rewards paid into.

## The quest log

++j++ calls `HarvestGame.ShowQuestLog()`, which is a four-row `ShowMenu` rather
than a bespoke panel:

```csharp
public void ShowQuestLog()
{
    HarvestHudRef.ShowMenu(Lexicon.Get("quest.title", "QUEST LOG"),
        new[]
        {
            SagaSystem.CurrentLine,
            SagaSystem.CompletedLine,
            SagaSystem.ProgressLine,
            Lexicon.Get("campaign.close", "Close")
        }, "Esc to close", _ => { });
}
```

| Row | Built by | Reads like |
|---|---|---|
| Current | `SagaSystem.CurrentLine` | `▶ Plant a Rose Garden` |
| Completed | `SagaSystem.CompletedLine` | `✓ Plant a Rose Garden  •  Hold a hymn…` |
| Progress | `SagaSystem.ProgressLine` | `Campaign 1/3   •   Devotion 14   •   92 rooms / ~95 min` |
| Close | `campaign.close` | `Close` |

`CompletedLine` returns `✓ none yet` while `Stage <= 0` and joins the finished
titles with `  •  `. `ProgressLine` clamps `campaignStage + 1` to 3, so it never
reads `4/3`. `CurrentTitle` looks up `Keys[Stage]` with a clamped index, which
means an out-of-range stage shows the last key instead of blank text.

The Oracle gives the same information in dialogue. `TalkToNpc` picks
`QuestHintKeys[Stage]` — one `npc.hint.*` line per stage — and appends
`npc.progress` once at least one Warden is down.

## Campaign stages and act intros

`campaignStage` is not a quest counter: it is the number of Wardens broken,
0-3. `HarvestGame.CampaignStage` clamps it, and both the map and the HUD read
that clamp.

| Act | Map row (`campaign.act1`…) | Warden | Rooms | Devotion |
|---|---|---|---|---|
| 0 | `ACT I  •  ASHEN ORCHARD  •  The Gilded Stag` | The Gilded Stag | 20 | 8 |
| 1 | `ACT II  •  DROWNED CONSERVATORY  •  The Mire Queen` | The Mire Queen | 32 | 12 |
| 2 | `ACT III  •  EMBER CITADEL  •  The Obsidian Sultan` | The Obsidian Sultan | 40 | 16 |

`JourneyPacing` totals the three acts at 92 rooms and targets 80-120 minutes
(`ShortSummary` reports `92 rooms / ~95 min`, which is what the quest log and
status line show).

++c++ opens `ShowCampaignMap()` in the hub. Each act row is labelled `CLEARED`,
`AVAILABLE` or `LOCKED` from its index against `CampaignStage`; choosing a
locked act toasts `campaign.locked.toast` ("Clear the previous warden first")
instead of starting it. The remaining rows are the devotion upgrades, the quest
log, and Close.

Picking an available act runs `ShowActIntro(act)`, which sets `currentAct`,
builds `act.intro` — `"The road to {0} opens. {1} waits at its end."` — and
offers two choices: Begin pilgrimage (`EnterExpedition(currentAct)`) or Return
to the Harvesthold.

## The victory sequence

Victory is a flag check on the way home, not a scripted cutscene. The order is:

1. The third Warden dies and `Dungeon` calls `WardenDefeated(2, bossName)`,
   which sets `campaignStage = 3`, pays 16 devotion (18 with the Crown) and
   calls `SagaSystem.Evaluate`.
2. The player leaves the expedition with loot; `ExitExpedition(true)` plays the
   portal, returns to the hub and calls `HarvestSaveSystem.Autosave()`.
3. Immediately after, `ExitExpedition` checks
   `campaignStage >= 3 && !victorySeen` and calls `ShowVictory()`.
4. `ShowVictory()` sets `victorySeen = true`, adds 10 devotion and opens the
   `campaign.victory.title` menu.

```csharp
void ShowVictory()
{
    HarvestState.Data.victorySeen = true;
    HarvestState.Data.devotion += 10;
    HarvestState.NotifyChanged();
    HarvestHudRef.ShowMenu(Lexicon.Get("campaign.victory.title", "THE HARVEST ENDURES"),
        new[]
        {
            Lexicon.Get("campaign.victory.body", "All three wardens have fallen. …"),
            Lexicon.Get("campaign.victory.reward", "The harvest is eternal."),
            Lexicon.Get("campaign.close", "Close")
        }, "Press 1-3", _ => { });
}
```

Because `victorySeen` is saved, the screen shows once per save. The autosave
runs *before* the flag is written, so the flag lands in the file on the next
save (or the next successful expedition); this is harmless because `ShowVictory`
is also gated on the live flag.

## Quest state in the save

There is no quest file and no quest component. The whole chain is three fields
on `HarvestSaveData`, serialised with the rest of the state by
`HarvestSaveSystem`:

| Field | Type | Default | Meaning |
|---|---|---|---|
| `questStage` | `int` | `0` | current stage, 0-8 |
| `campaignStage` | `int` | `0` | Wardens broken, 0-3 |
| `victorySeen` | `bool` | `false` | victory screen already shown |
| `hymns` | `int` | `0` | Hymns held; also gates stage 1 |
| `devotion` | `int` | `0` | reward currency and stage 7 gate |

`HarvestState.Reset()` (called by `StartNewGame`) replaces `Data` with a fresh
`HarvestSaveData`, which is the only thing that rewinds the chain.

!!! warning "Hub quests need a save"

    Autosave fires on a successful expedition return, not on quest completion.
    Farm, hymn, temple, companion and devotion stages finish in the hub and stay
    in memory until the next ++f5++ or expedition. Tell players to save before
    quitting after a hub milestone.

## Adding your own stage

The chain is data-light by design, so a new stage is four edits and one
localization entry. This walkthrough appends a tenth stage that asks for a Bell
Tower.

**1. Raise `FinalStage` and add the key.** `FinalStage` is the highest valid
stage, not the number of stages.

```csharp
public const int FinalStage = 9;

static readonly string[] Keys =
{
    "quest.buildfarm",
    // …
    "quest.devotion",
    "quest.done",
    "quest.belltower"
};
```

**2. Add the condition arm.** Stage numbers must stay contiguous from 0.

```csharp
7 => HarvestState.Data.devotion >= 30,
8 => HasBellTower(boot),
_ => true
```

**3. Write the predicate.** A `Sanctum` lookup is the cheapest pattern:

```csharp
static bool HasBellTower(HarvestGame boot)
{
    foreach (var building in boot.Sanctums)
        if (building != null && building.Type == SanctumType.BellTower) return true;
    return false;
}
```

**4. Add the Oracle hint.** `QuestHintKeys` in `GameBootstrap.cs` must grow in
lockstep or the Oracle will repeat the wrong line:

```csharp
static readonly string[] QuestHintKeys =
{
    "npc.hint.farm", "npc.hint.hymn", "npc.hint.reflection",
    "npc.hint.forest", "npc.hint.temple", "npc.hint.sanctum",
    "npc.hint.companions", "npc.hint.devotion", "npc.hint.done",
    "npc.hint.belltower"
};
```

**5. Add the strings** to both `en.json` and `xx.json`:

```json
{ "key": "quest.belltower", "value": "Raise a Bell Tower" },
{ "key": "npc.hint.belltower", "value": "A Bell Tower would call the flock home. Build one." }
```

!!! warning "Two silent failure modes"

    - If you forget to raise `FinalStage`, the new arm is unreachable: the
      `while` loop stops at the old maximum.
    - If you forget the `Keys` entry, `CurrentTitle` clamps to
      `Keys.Length - 1` and quietly shows the previous stage's title. Both
      mistakes look like "the quest never finishes" in play.

If the new stage needs state of its own, add the field to `HarvestSaveData` and
follow the schema-bump walkthrough in [Saving and loading](save-system.md#adding-a-field-a-worked-example)
so old saves get a sensible default.

## Related

- [Saving and loading](save-system.md) — how `questStage` reaches disk
- [Expeditions](expeditions.md) — where `campaignStage` is raised
- [The Harvesthold hub](hub-and-companions.md) — farms, temples and the flock
- [Rites and sanctums](../reference/catalogs.md) - the altar ceremony behind stage 1
