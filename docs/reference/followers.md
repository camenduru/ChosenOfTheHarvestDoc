# Followers API reference

Companions are runtime `CompanionAgent` objects bound to a persistent
`CompanionRecord`: the record survives save/load, the agent does the walking,
working and socialising. This page documents both, the job/bed/priority plumbing
they share with `Sanctum`, and the two small helpers that make the hub readable —
`NavGrid` for pathing and `CompanionNameTag` for billboarded names.

## At a glance

`CompanionAgent` is the brain and body, `CompanionRecord` is the save data,
`CompanionCatalog` supplies names/traits/tints, `NavGrid` is a grid A* over the hub,
and `CompanionNameTag` is the floating label. All five live in the
`ChosenOfTheHarvest` namespace.

!!! note "Records are index-aligned with agents"
    `HarvestState.CompanionAt(index)` and `HarvestGame.Companions[index]` describe the
    same companion. `SpawnCompanion` binds them in order, and friendship/partnership
    fields store those indices, so never reorder the companion list without a save
    migration.

## CompanionRecord

Plain `[Serializable]` data in `Scripts/Core/GameState.cs`. Defaults matter: a new
record starts at 100 hunger, 70 loyalty, level 1 and age 18–36.

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Unique display name |
| `trait` | `string` | `CompanionTrait` name, parsed case-sensitively |
| `age` | `int` | 0–90; under 18 means child |
| `level`, `xp` | `int` | Level 1–5, XP toward the next level |
| `hunger` | `float` | 0–100, decays every frame |
| `loyalty` | `float` | 0–100; under 20 means dissenting |
| `exhaustion` | `float` | 0–100; over 85 forces sleep |
| `sick` | `bool` | Illness flag |
| `mealBuff` | `float` | Seconds of fed bonus, capped at 180 on load |
| `tintR`, `tintG`, `tintB` | `float` | Body tint (white when the name has portrait art) |
| `priority` | `int` | 0 work, 1 grace, 2 social |
| `job` | `int` | Job sanctum index, `-1` for none |
| `bed` | `int` | Assigned bed index, `-1` for none |
| `inReflection` | `int` | Reflection chamber index, `-1` for none |
| `friend`, `partner` | `int` | Companion indices, `-1` for none |
| `parentA`, `parentB`, `childDay` | `int` | Lineage fields, `-1` for none |

## CompanionAgent

`MonoBehaviour` in `Scripts/Followers/FollowerAgent.cs`. `Speed` is the only public
field (2.2 by default); everything else is a read-only view over the bound record
plus the live `Mode`.

**Identity and needs**

| Member | Type | Description |
|---|---|---|
| `Record` | `CompanionRecord` | Bound save data; `null` until `BindRecord` |
| `Trait` | `CompanionTrait` | Parsed from `Record.trait` |
| `CompanionName` | `string` | `Record.name`, or `"Companion"` when unbound |
| `Hunger`, `Loyalty`, `Exhaustion` | `float` | Record values, with 100/70/0 fallbacks |
| `Sick` | `bool` | `Record.sick` |
| `Dissenter` | `bool` | `Loyalty < 20` |
| `Age`, `Level` | `int` | From the record |
| `IsChild` | `bool` | `Age < 18` |
| `Mood` | `string` | Localised mood, see the thresholds below |
| `StatusLine` | `string` | Mood, trait, job and `H0 L0 E0` summary |
| `RelationshipLine` | `string` | Partner, friend or parent line, else empty |
| `WorkBonus` | `int` | Hardworking 2 else 1, plus `(Level - 1) / 2`, plus 1 while fed |
| `GraceBonus` | `int` | 2 for Devout, else 0 |

**Task state**

| Member | Type | Description |
|---|---|---|
| `CurrentMode` | `Mode` | Live behaviour mode |
| `Job`, `Bed` | `Sanctum` | Assigned workplace and bed |
| `Priority` | `int` | 0–2, clamped from the record |
| `PriorityName` | `string` | `Work`, `Grace` or `Social` |
| `CanWork` | `bool` | Not sick/dissenting, adult, and not in a blocking mode |
| `IsSleeping` | `bool` | Mode is `Sleep` or `Rest` |
| `InReflection` | `bool` | Mode is `Reflection` |
| `Arrived` | `bool` | Set by `Attend` once within 0.4 units of its point |

**Commands**

| Method | Description |
|---|---|
| `Init(Vector3 homePosition, Color tint)` | Grounds the agent at home and starts wandering |
| `BindRecord(CompanionRecord record)` | Binds save data and refreshes tint |
| `Attend(Vector3 point)` | Walks to a point and reports `Arrived`; ignores the decision loop |
| `WorkAt(Sanctum building)` | Claims `building.Worker`, targets `WorkSpot`, enters `Work` |
| `AssignJob(Sanctum building)` | Alias of `WorkAt` |
| `AssignBed(Sanctum bed)` | Claims `bed.Sleeper` and releases the previous bed |
| `ClearJob()` | Releases the workplace and returns to wandering |
| `WanderHome()` | Targets the job or home, whichever applies |
| `Feed()` | Hunger 100, +6 loyalty, 30 s meal buff |
| `EatFood()` | +55 hunger, +1 loyalty |
| `EatMeal()` | +80 hunger, +4 loyalty, 90 s meal buff |
| `Treat()` | Clears illness, -20 exhaustion, +5 loyalty |
| `AddLoyalty(float amount)` | Clamps 0–100 |
| `SetPriority(int value)` | Clamps 0–2 and toasts the new priority |
| `AddXp(int amount)` | Adds XP and levels up while `xp >= 50 * level` |
| `Bless()` | +15 loyalty, -25 exhaustion, cures illness |
| `RestNow()` | Enters `Rest` for 12 s at the bed or home |
| `EnterReflection(Sanctum reflection)` | Returns `false` unless the chamber is free and `IsReflection` |
| `ReleaseFromReflection()` | Frees the chamber and returns to wandering |
| `FlushHaul()` | Deposits carried produce before a save or a forced task change |

## Mode and the decision loop

`Mode` has twelve values; the agent re-decides every 0.8 s.

| Mode | Entered by | Left when |
|---|---|---|
| `Wander` | Default, dissent, illness, no job | A higher rule applies |
| `Attend` | `Attend` (cutscenes, ceremonies) | Explicitly re-tasked |
| `Work` | `WorkAt`, `WanderHome`, wake-up | Night, dissent, illness, no job |
| `Sleep` | Night with a bed, exhaustion ≥ 85 | Not night and exhaustion < 30, or ≤ 4 |
| `Eat` | Hunger < 25 with food in store | Reaches `FoodSpot` |
| `Rest` | Night without a bed, exhaustion, `RestNow` | Exhaustion < 20 or the rest timer expires |
| `Haul` | Job store full at the work spot | Reaches `HaulSpot` |
| `Clean` | No job and a shrine with `Dirt > 25` | `Dirt ≤ 4` |
| `Reflection` | `EnterReflection` | Loyalty ≥ 45 or 40 s of reflection |
| `Repair` | No job and a sanctum with `Condition < 60` | `Condition ≥ 99.5` |
| `Social` | Dusk with priority 2 and a Tea House | Dusk ends, sick, dissenting |
| `Pray` | Dusk with priority 1 and a Shrine | Dusk ends, sick, dissenting |

The order below is the verified order in `Decide()`; the first match wins.

1. `Reflection` releases at loyalty ≥ 45 or after 40 s.
2. `Social`/`Pray` drop to `Wander` outside dusk, or when sick or dissenting.
3. `Sleep` wakes outside night once exhaustion < 30, or at ≤ 4.
4. `Rest` wakes at exhaustion < 20 or when its timer runs out.
5. `Eat` and `Haul` make no decisions until they arrive.
6. Night: `Sleep` if a bed is assigned, otherwise `Rest`.
7. Exhaustion ≥ 85 with a bed forces `Sleep` even by day.
8. Hunger < 25 with food available sends the agent to `Eat`.
9. Dissenters and the sick are forced to `Wander`.
10. Dusk plus priority 2 sends them to the nearest Tea House (`Social`); priority 1 to the nearest Shrine (`Pray`).
11. Unfinished `Clean`/`Repair` continue while the target still needs work.
12. With no job: the nearest dirty shrine, then the nearest damaged sanctum.
13. With a job: `Work` at `Job.WorkSpot`.
14. Otherwise: `Wander` around home.

## Needs, sickness and loyalty

`TickNeeds` runs only while input is unlocked, so pausing the game does not starve
anyone. All rates are per real second.

| Need | Rate |
|---|---|
| Hunger | −0.30 base; Glutton ×1.5, `austerity` creed ×0.8, sick ×1.1, asleep ×0.3, reflecting ×0.55 |
| Starvation | 45 s at zero hunger calls `HarvestGame.CompanionDied` |
| Exhaustion | +0.10 awake (Lazy 0.15), ×1.35 sick; −3.2 asleep or resting, −0.9 reflecting |
| Loyalty | −0.065 while suffering, otherwise +0.012; the Loyal trait scales the rate ×0.6 |
| Social mode | +0.04 loyalty while socialising |

Suffering means hunger < 25, illness, or exhaustion > 90. On top of that, standing
inside a sanctum aura trickles loyalty: Shrine 0.018, Temple 0.02, Bell Tower 0.018,
Obelisk 0.015, Tea House 0.01, Sun Bath 0.01 (and −0.08 exhaustion), Reliquary 0.01,
plus `0.012 × InteriorAuraBonus` for built interiors.

Mood is a cascade: `Dissenting` (< 20 loyalty), `Ill`, `Starving` (< 20 hunger),
`Exhausted` (> 80), `Sleeping`, `Hungry` (< 45), `Tired` (> 60), `Devoted` (> 85),
then `Content`.

Illness is rolled every 12 s: 35 % chance for Sickly and 8 % otherwise, but only
while hunger < 25 or exhaustion > 85. Recovery needs hunger > 60, exhaustion < 35,
at least 18 s ill, and a small per-frame roll. Work grants 1 XP per second, levels
cap at 5, and each level gives +4 loyalty.

## Jobs, beds and priorities

`WorkAt` claims the sanctum's single `Worker` slot and releases the previous one;
`AssignBed` does the same for `Sleeper`. Companions touch only these `Sanctum`
members (see `Scripts/Buildings/Building.cs`):

| Member | Type | Description |
|---|---|---|
| `Worker`, `Sleeper`, `Penitent` | `CompanionAgent` | Exclusive occupants |
| `WorkSpot`, `SleepSpot`, `ReflectionSpot` | `Vector3` | Offsets from the building |
| `IsReflection` | `bool` | True for the Reflection Chamber |
| `Type` | `SanctumType` | Aura, job and priority rules key off this |
| `Stored` | `int` | Produced units waiting for a haul |
| `Dirt`, `Condition` | `float` | Cleaning and repair thresholds |
| `TakeStored()` | `int` | Removes and returns the stored units |

While working, an agent hauls as soon as the store reaches 4 units (2 for a Herb
Garden), carrying them to `HarvestGame.HaulSpot`. Priorities change daily life, not
the work loop: priority 1 adds +1 grace every 20 s, priority 2 socialises harder at
dusk. `HarvestGame.RecruitCompanion` refuses at 30 companions, and
`HarvestGame.CompanionDied` removes the agent and leaves a memorial.

## Relationships

Friendship is attempted every 12 s with a Tea House (18 s without) for companions
who are not sleeping, reflecting, or already paired: both must be within 5 units
(3 without a Tea House), then both gain +4 loyalty and each records the other.
Romance is attempted every 30 s and needs both sides at age ≥ 18, loyalty ≥ 80 and
level ≥ 2, plus an 18 % roll; success grants +10 loyalty each. `EnterReflection`
gives dissenters a route back: they leave their job, sit in the chamber, and exit
reformed at loyalty 45 or after 40 s.

## CompanionCatalog

`Scripts/Followers/FollowerCatalog.cs`.

| Member | Returns | Description |
|---|---|---|
| `InnerCircleCount` | `const int` | 8 — companions with dedicated portrait art |
| `InnerCircleName(int index)` | `string` | Name for 0–7, otherwise `null` |
| `Traits` | `CompanionTrait[]` | All six traits in enum order |
| `RandomName()` | `string` | Uniform pick from the 30-name pool |
| `RandomTrait()` | `CompanionTrait` | Uniform pick from `Traits` |
| `TraitName(CompanionTrait trait)` | `string` | `Lexicon.Get("trait.<lower>", …)` |
| `TraitDescription(CompanionTrait trait)` | `string` | `trait.<lower>.desc` |
| `Parse(string id)` | `CompanionTrait` | Case-sensitive name match; unknown ids become `Loyal` |
| `RandomTint()` | `Color` | Rose or amber blended up to 25 % toward teal |

**Inner circle** — `Art/Characters/<lowercase name>` is loaded first, so these have
their own generated portraits (Amara reuses `companion.png`):

| Index | Name | Portrait sprite |
|---|---|---|
| 0 | Amara | `companion` |
| 1 | Sable | `sable` |
| 2 | Iris | `iris` |
| 3 | Nadia | `nadia` |
| 4 | Yara | `yara` |
| 5 | Liora | `liora` |
| 6 | Mira | `mira` |
| 7 | Yafit | `yafit` |

The remaining pool names are Calista, Delphine, Esme, Faye, Gemma, Helia, Isolde,
Jade, Kira, Lyra, Maeve, Nerissa, Opal, Perla, Quinn, Rhea, Seraphine, Talia, Uma,
Vesper, Wren and Yvonne; they share `companion.png` and rely on their tint to stand
apart. `CompanionTrait` is `Loyal`, `Hardworking`, `Glutton`, `Devout`, `Sickly`,
`Lazy`.

| Trait | Description | Mechanical effect |
|---|---|---|
| `Loyal` | loyalty fades slowly | Loyalty rate ×0.6, slowing both gain and loss |
| `Hardworking` | steadier worker, +2 output | `WorkBonus` base is 2 |
| `Glutton` | hungers quickly | Hunger rate ×1.5 |
| `Devout` | +2 grace at hymns | `GraceBonus` 2 |
| `Sickly` | falls ill easily | Illness chance 35 % instead of 8 % |
| `Lazy` | works slowly, tires quickly | Move speed ×0.85, exhaustion +0.15/s |

## NavGrid

Static grid A* over the hub. `Begin` allocates the grid, `Mark` blocks scenery,
`FindPath` returns a corner-clipped path or `null`.

| Member | Returns | Description |
|---|---|---|
| `Ready` | `bool` | True between `Begin` and `Clear` |
| `Begin(Vector2 halfBounds, float cell = 1f)` | `void` | Allocates the grid; cell size is clamped to ≥ 0.5 |
| `Mark(Vector3 world, float radius)` | `void` | Blocks every cell within `radius + 0.42` |
| `End()` | `void` | Leaves `Ready` set — it does not post-process the grid |
| `Clear()` | `void` | Drops the grid and clears `Ready` |
| `FindPath(Vector3 from, Vector3 to)` | `Vector3[]` | Cell centres excluding the start cell; `null` if unreachable |

Paths are 8-directional with diagonal corner-cutting disallowed, use the octile
heuristic, and snap a blocked goal to the nearest free cell within 4 cells. Agents
repath at most every 0.9 s and fall back to a straight line when `Ready` is false,
then clamp themselves to `HarvestGame.Instance.Player.Bounds`.

## CompanionNameTag

`Create(Transform parent, Transform target, string text, Color tint)` builds a
`TextMesh` label 2.35 units above the target. `LateUpdate` copies the main
camera's rotation, scales with distance (clamped 0.9–1.2), renders at sorting
order 5000, and destroys itself if the target disappears.

```csharp
var boot = HarvestGame.Instance;
var companion = boot.Companions[0];

// Point the first companion at a shrine job and a bed, then read its state.
Sanctum shrine = null, bed = null;
foreach (var sanctum in boot.Sanctums)
{
    if (sanctum.Type == SanctumType.Shrine) shrine = sanctum;
    else if (sanctum.Type == SanctumType.Bed) bed = sanctum;
}

if (shrine != null) companion.AssignJob(shrine);
if (bed != null) companion.AssignBed(bed);
companion.SetPriority(1);            // 0 work, 1 grace, 2 social

Debug.Log(companion.StatusLine);     // mood, trait, job, H/L/E
Debug.Log(companion.Mood + " / " + companion.PriorityName);
```

!!! tip "Testing the loop"
    `HarvestGame.Companions` is a read-only list, so a test can walk every agent,
    force `SetPriority`, `Feed` or `RestNow`, and log `CurrentMode`. The built-in
    smoke test does exactly this before it exercises the companion menu.

## Related

- [Dungeon API](dungeon.md) — rescue rooms call `HarvestGame.RecruitCompanion`
- [The Harvesthold hub](../guide/hub-and-companions.md) — player-facing companion flow
- [Core API](core.md) — `HarvestState`, `CompanionRecord` and save schema
- [Building & sanctums](../guide/building.md) — the workplaces companions claim
