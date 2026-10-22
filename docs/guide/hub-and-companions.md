# The Harvesthold hub and companions

The Harvesthold is the safe half of *Chosen of the Harvest*: a 40 by 30 metre
ground plane with an altar, a statue, a portal, one talkative NPC and a court of
companions who need food, sleep and reasons to stay. You return here after every
pilgrimage, and almost every system you touch outside the Fallow — building,
ceremonies, the roster — is anchored to a single object in the scene. This page
covers the physical layout, the day loop, and the full companion simulation
behind the `CompanionRecord` class.

!!! note "One class owns the hub"
    `HarvestGame` (`Scripts/Core/GameBootstrap.cs`) builds the hub in `Awake()`
    from code — there are no prefabs and no scene references. `BuildGround`,
    `BuildHarvestAltar`, `BuildPlayer`, `BuildResources`, `BuildCompanions`,
    `BuildPortal` and `BuildNpc` run in that order.

## The hub at a glance

| Landmark | Position (x, z) | What it does |
|---|---|---|
| Harvest Altar | `0, 1.5` | Ceremony menu, 30 s cooldown, radius 3.2 |
| Temple statue | `0, 8` | Backdrop; blocks building placement |
| Pilgrimage portal | `8.2, -4.5` | Opens the pilgrimage map (act select) |
| The Oracle | `4, 3.8` | Nine-step hint dialogue and campaign progress |
| The Sovereign (player) | `0, -3` | Spawn point; returns here after death |
| Gathering nodes | random | 10 trees, 8 rocks, three hits each |

Movement is clamped to `hubBounds`, computed once as
`(WorldWidth / 2 - 1, WorldHeight / 2 - 1)` — about `±19` on x and `±14` on z.
The camera follows the player, clamped a little tighter still
(`WorldWidth / 2 - 6`). Building placement uses its own tighter box
(`±17.5` by `±12.5`).

```csharp
void BuildHarvestAltar()
{
    var altarSprite = Resources.Load<Sprite>("Art/World/altar");
    Vector3 altarPos = Stage3D.P(0f, 1.5f);
    var altarGo = Stage3D.CreateBillboard(hubRoot.transform, "HarvestAltar", altarSprite, altarPos,
        2.6f, Color.white, Stage3D.SortOrder(altarPos));
    Stage3D.CreateShadow(hubRoot.transform, altarGo.transform, 1.3f);

    HarvestAltarRef = altarGo.AddComponent<HarvestAltar>();
}
```

## The Harvest Altar

`HarvestAltar` is deliberately tiny: a radius, a cooldown and a `CanPreach`
gate. Standing inside 3.2 metres and pressing ++e++ opens the ceremony menu;
`MarkUsed()` starts a 30-second rest during which the altar refuses every
ceremony. Companions physically walk to seats arranged on an ellipse around it
before the effect resolves.

See [Ceremonies and omens](ceremonies.md) for the five ceremony types and the
grace maths.

## The pilgrimage portal

The portal is a door sprite with a purple glow and a floating `PILGRIMAGE`
label. Interacting with it does **not** drop you into the Fallow immediately — it
opens the pilgrimage map, which lists the three acts, the devotion upgrade
screen and the quest log. Choosing an available act shows that act's intro card,
and only `Begin pilgrimage` calls `EnterExpedition(act)`.

!!! warning "The hub keeps running"
    `EnterExpedition` deactivates `hubRoot` rather than unloading the scene, so
    building timers, companion needs and the day cycle pause while you are in
    the Fallow. `SeasonCycle.Update` and `Sanctum.Update` both early-out on
    `HarvestGame.InputLocked`.

## The Oracle

There is exactly one NPC in the hub. `BuildNpc` loads
`Art/Characters/oracle`, tints it lavender and hangs a `CompanionNameTag` reading
`THE ORACLE`. Pressing ++e++ within 2.2 metres calls `TalkToNpc()`, which walks a
short intro on the first visit and then hands out one hint per
`QuestHintKeys` entry — build a rose garden, hold a hymn, break the first warden,
and so on. The ninth line is the closing speech.

There is no merchant in the hub. The merchant is a dungeon fixture that appears
in `RoomType.Shop` rooms only; see [Expeditions](expeditions.md#shop-rooms).

## Day and season cycle

`SeasonCycle` (`Scripts/Core/DayCycle.cs`) is a four-phase clock: **Dawn**,
**Day**, **Dusk**, **Night**, 60 real seconds each. One full day is therefore four
minutes. Each phase repaints `Camera.main.backgroundColor`; Night is the only
phase companions treat as bedtime.

At the transition *into* Dawn the day counter increments, and `HarvestGame`
runs the nightly bookkeeping:

| Step | Rule |
|---|---|
| Ageing | every companion `age++` |
| Old age | from age 55, death chance `(age - 54) × 1%` per day |
| Births | `TryBirths()`, one attempt per dawn |
| Daily event | 45% chance of `TriggerDailyEvent()` |

The five daily events are: `+6 food`, `+10 pearls`, `+8 grace`, a storm that
takes 18 condition from the most damaged sanctum (or `+4 wood` if nothing is
damaged), and `+2 loyalty` to everyone (or `+5 grace` with no companions).

```csharp
timer += Time.deltaTime;
if (timer < PhaseSeconds) return;

timer -= PhaseSeconds;
PhaseIndex = (PhaseIndex + 1) % Phases.Length;
if (PhaseIndex == 0) HarvestState.Data.day++;
```

## Recruiting companions

A new save starts with `companions = 4`. Four routes add more, all capped at 30:

- **Build a bed.** `CreateSanctumAt` increments `Data.companions`, spawns the
  agent and assigns the bed index to the new record.
- **Rescue a captive.** A `Rescue` room calls `RecruitCompanion()` when you free
  the cage. At the cap the room refuses with "The harvest is at its companion cap".
- **Births.** Two partners with loyalty ≥ 80 and age ≥ 18 may produce a child at
  dawn (18% chance, at least three days apart, one birth per dawn).
- **Narrative hooks.** `RecruitCompanion()` is public, so quests and cutscenes can
  call it directly.

## The companion record

Every runtime `CompanionAgent` binds to a serializable `CompanionRecord`. The
agent owns movement and animation; the record owns anything that must survive a
save.

| Field | Type / default | Meaning |
|---|---|---|
| `name` | `""` | Unique; first eight come from the inner circle |
| `trait` | `"Loyal"` | One of six `CompanionTrait` values |
| `tintR/G/B` | `0.85, 0.75, 0.95` | Sprite tint so generated companions differ |
| `hunger` | `100` | 0–100, falls 0.30/s |
| `loyalty` | `70` | 0–100; below 20 the companion is a dissenter |
| `exhaustion` | `0` | 0–100, rises 0.10/s awake |
| `sick` | `false` | Blocks work |
| `bed` | `-1` | Index into `Data.buildings` |
| `job` | `-1` | Index into `Data.buildings` |
| `priority` | `0` | 0 Work, 1 Grace, 2 Social |
| `inReflection` | `-1` | Index of the reflection chamber holding them |
| `mealBuff` | `0` | Seconds of fed bonus remaining (max 180) |
| `friend` | `-1` | Mutual index pair |
| `partner` | `-1` | Mutual index pair; prerequisite for children |
| `parentA`, `parentB` | `-1` | Biological parents, used for inheritance |
| `childDay` | `-1` | Day of last child, throttles births |
| `age` | `18–35` | Years; incremented at dawn |
| `level`, `xp` | `1`, `0` | Levels 1–5, `50 × level` xp per level |

```csharp
public static CompanionRecord CreateCompanionRecord()
{
    var tint = CompanionCatalog.RandomTint();
    return new CompanionRecord
    {
        name = UniqueCompanionName(),
        trait = CompanionCatalog.RandomTrait().ToString(),
        tintR = tint.r, tintG = tint.g, tintB = tint.b,
        hunger = 100f,
        age = UnityEngine.Random.Range(18, 36),
        loyalty = 70f + (Data != null ? Data.metaCharisma * 3 : 0),
        exhaustion = 0f, bed = -1, job = -1
    };
}
```

`EnsureCompanionRecords()` is the repair pass. It runs on every load, every
recruit and every death: it trims or pads the record list to match
`Data.companions`, fills missing names and traits, clamps every need, repairs
records from save schema versions 1 and 2, and renames duplicates.

## Traits

`CompanionCatalog.Traits` holds six entries. Traits are rolled at random when a
record is created and never change afterwards.

| Trait | Effect in code |
|---|---|
| Loyal | loyalty decay multiplied by `0.6` |
| Hardworking | `WorkBonus` starts at 2 instead of 1 |
| Glutton | hunger drain × `1.5` |
| Devout | `GraceBonus` +2 at hymns |
| Sickly | illness chance `35%` per check instead of `8%` |
| Lazy | exhaustion rises `0.15/s` instead of `0.10/s` |

`WorkBonus` is the number that feeds building output:
`(Hardworking ? 2 : 1) + (level - 1) / 2 + (mealBuff > 0 ? 1 : 0)`.

## Needs and behaviour states

Companions re-decide every `0.8 s` in `Decide()`. The agent has twelve modes:

| Mode | Entered when |
|---|---|
| `Wander` | Nothing else applies; picks a random point near home |
| `Attend` | Called by a ceremony; walks to a seat and sets `Arrived` |
| `Work` | Has a `Job`; stands at `Job.WorkSpot` |
| `Sleep` | Night with a bed, or exhaustion ≥ 85 |
| `Rest` | Night without a bed, or `RestNow()` |
| `Eat` | Hunger < 25 and food or meals exist |
| `Haul` | The assigned workplace has ≥ 4 stored (2 for herb gardens) |
| `Clean` | Idle and a shrine is dirtier than 25 |
| `Repair` | Idle and a sanctum is below 99.5 condition |
| `Reflection` | Confined in a reflection chamber |
| `Social` | Dusk, priority Social, and a tea house exists |
| `Pray` | Dusk, priority Grace, and a shrine exists |

The numbers in `TickNeeds` are the whole survival loop:

- Hunger falls `0.30/s`; sleeping slows it to 30%, reflection to 55%.
- At hunger 0 a 45-second starvation timer starts; when it expires the companion
  dies and calls `CompanionDied(this)`.
- Exhaustion rises `0.10/s` awake and falls `3.2/s` asleep. Sleeping ends when it
  drops below 30 or it is no longer night.
- Loyalty drifts `+0.012/s`, or `-0.065/s` while hungry, ill or exhausted past 90.
- Illness is checked every 12 s when hunger < 25 or exhaustion > 85.
- Working earns `1 xp/s`.

A companion cannot work while sick, dissenting, under 18, or in any mode other
than idle/work — that is the `CanWork` property. Children therefore wander and
eat but never staff a building.

## Feeding, rest and healing

| Action | Cost | Effect |
|---|---|---|
| `Feed()` (menu, food) | 1 food | hunger 100, +6 loyalty, 30 s fed buff |
| `EatFood()` (automatic) | 1 food | hunger +55, +1 loyalty |
| `EatMeal()` (automatic or menu) | 1 meal | hunger +80, +4 loyalty, 90 s fed buff |
| `RestNow()` | free | Rest for 12 s at the bed |
| `Treat()` | 1 medicine | cures illness, −20 exhaustion, +5 loyalty |
| `Bless()` | 5 grace | −25 exhaustion, +15 loyalty, cures illness |

Meals are always eaten before raw food, both in the automatic `Eat` branch
(`ConsumeFoodFor`) and in the roster's Feed action.

## Illness and dissent

Sickness is a state, not a timer: it persists until `Treat()`, `Bless()`,
`CeremonyType.Warding`, or a natural recovery roll. Recovery needs hunger > 60
and exhaustion < 35, then ticks at roughly 1% per second after 18 seconds ill.

Dissent is simply `Loyalty < 20`. A dissenter stops working, refuses to socialise,
and toasts once ("is losing grace"). Two things fix it: time in a
**reflection chamber**, which adds `+0.14 loyalty/s` and releases at 45 loyalty
or after about 40 seconds, or `Bless()` / `Warding` / a funeral.

## Relationships, children and ageing

`TrySocialize()` runs every 18 s, or every 12 s when a tea house stands, and
pairs two unpartnered companions within 3 m (5 m with a tea house) as friends.
`TryRomance()` runs every 30 s and upgrades a mutual friendship to partners.
Both writes are symmetric: `A.friend == B` and `B.friend == A`.

`TryBirths()` requires mutual partners, both aged 18+, both loyalty ≥ 80, three
days since either partner's last child, and an 18% roll. The child is appended to
the record list with `age = 0`, loyalty 85, and a tint averaged from both
parents. Children age one year per in-game day and become workers at 18.

When a parent dies, the eldest child present inherits the bed and the job if both
are free — `CompanionDied` looks for `parentA`/`parentB` matches before creating
the memorial.

## Death and memorials

A `MemorialData` is appended for every death, with the name, the day, and a
position: inside a `SanctumType.Memorial` aura if one exists, otherwise wherever
the companion fell. Surviving companions lose 8 loyalty, and every relationship
index pointing at the dead companion is cleared or shifted down.

Interacting with a grave offers:

| Choice | Cost | Effect |
|---|---|---|
| Mourn | free, once | +10 grace, +5 loyalty to every survivor |
| Hold a funeral | 5 food + 10 grace, once | +12 loyalty and −12 exhaustion to everyone |
| Remember | free | Flavour text; the grave stays named |

## The roster screen

Press ++f++ in the hub. `ShowCompanionRoster` pages five companions at a time and
prints one line each:

```
Amara   •   Lv1 A28   •   Content   •   Rose Garden   •   H100 L70
```

Choosing a name opens `ShowCompanionActions`, six entries: **Assign job**,
**Feed**, **Treat**, **Duty**, **Bed / Rest**, and **Reflection**. Duty holds the
three priorities plus Bless. Bed lists "Rest here now" and the first four beds,
with the current sleeper shown; taking an occupied bed evicts the previous owner.

## The generated companion pool

`CompanionCatalog` ships 30 names. The first eight — Amara, Sable, Iris, Nadia,
Yara, Liora, Mira, Yafit — are the inner circle: they are assigned first, and each
has portrait art under `Resources/Art/Characters/`. Amara intentionally reuses the
shared `companion.png`. Every later companion draws from the full pool with a
check for duplicates, and falls back to the generic sprite plus the record tint.

## Under the hood

The agent binds the record once, then mirrors every write back through
`SyncRecord()`:

```csharp
public void BindRecord(CompanionRecord record)
{
    Record = record;
    Trait = CompanionCatalog.Parse(record == null ? "Loyal" : record.trait);
    UpdateAppearance();
}
```

Auto-assignment happens at build time, not by the player: `CreateSanctumAt` calls
`TryAssignWorker(built)`, and `AssignWorkers()` re-fills every empty workplace
after a load or a removal. `AssignBeds()` does the same for beds. A companion who
loses a building through `RemoveSanctum` is explicitly unassigned, and every
stored index is shifted down.

!!! tip "Extending the court"
    Because `CompanionRecord` is a plain `[Serializable]` class, adding a field is
    a two-step job: add it to the class, then clamp or default it inside
    `EnsureCompanionRecords`. The save system does not need to know about it —
    `JsonUtility` serialises the whole object graph.

## Related

- [Ceremonies and omens](ceremonies.md) — hymns, feasts, sacrifices and the altar cooldown
- [Building and sanctums](building.md) — beds, workplaces, reflection chambers and the interior grid
- [Economy and resources](economy.md) — where the food and medicine in this page come from
- [Combat and Wardens](combat.md) — what happens to the Sovereign outside the hub
