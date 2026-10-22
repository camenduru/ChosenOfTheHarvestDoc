# Building the Harvesthold

Every permanent improvement in *Chosen of the Harvest* is a sanctum: a billboard
placed on a one-metre grid around the hub altar. Sanctums produce food, wood,
stone, herbs, grace and pearls; they house workers and sleepers; and their auras
quiet or radicalise the companions standing nearby. This page covers the build
menu, placement rules, all twenty sanctum types, the condition and dirt model,
worker hauling, and the small interior grid you can decorate.

!!! note "Two keys, one system"
    ++b++ opens the build menu in the hub and the interior menu inside a
    building. ++k++ opens the cross-system catalog, which lists every sanctum
    with its cost and effect without entering placement mode.

## Placing a sanctum

`ShowBuildMenu` pages through `SanctumType` five entries at a time; choosing one
calls `BeginPlacement`. Placement is a separate input mode: the game locks
movement, spawns a translucent ghost, and waits for you to commit.

| Input | Action |
|---|---|
| ++w++ ++a++ ++s++ ++d++ | Move the ghost one metre at a time |
| ++q++ | Rotate a quarter turn (cosmetic flip) |
| ++e++ or ++space++ | Confirm |
| ++esc++ | Cancel and refund nothing, because nothing was spent |

The ghost is green when the spot is legal and affordable, red otherwise. Four
rules decide legality:

```csharp
if (Mathf.Abs(position.x) > hubBounds.x - 1.5f ||
    Mathf.Abs(position.z) > hubBounds.y - 1.5f)
    return false;

foreach (var building in buildings)
{
    if (building == null || building == ignore) continue;
    if (Vector3.Distance(position, building.transform.position) < 2.4f) return false;
}
```

The remaining checks reject the statue, altar, portal, NPC, gathering nodes and
the player's own body. Moving an existing sanctum uses the same ghost and the
same checks, but skips the affordability test — relocating is free, and workers
and sleepers are re-bound to the new spot on confirmation.

!!! warning "The hub box is tighter than the walkable box"
    You can walk to about ±19 by ±14, but placement is limited to the hub bounds
    minus 1.5 m. A sanctum also keeps 2.4 m of clear ground around it, so a
    dense row of farms is not possible. Plan lanes, not clusters.

## The twenty sanctums

Costs are charged in wood and stone at placement. `Aura` is the radius in metres
inside which the building's continuous effect applies; `—` means no aura.

| Sanctum | Wood | Stone | Height | Aura | Effect | Worker |
|---|---|---|---|---|---|---|
| Farm | 20 | 0 | 1.8 | — | +2 food / 12 s | Yes |
| Shrine | 0 | 25 | 2.6 | 4.2 | +1 grace / 8 s, scaled by dirt | Yes |
| Bed | 15 | 0 | 1.6 | — | A sleeping spot for one companion | No |
| Lumberyard | 30 | 0 | 2.8 | — | +1 wood / 10 s | Yes |
| Quarry | 20 | 20 | 2.2 | — | +1 stone / 12 s | Yes |
| Kitchen | 25 | 10 | 2.0 | — | 2 food → 1 meal / 16 s | Yes |
| Storage | 30 | 10 | 1.7 | — | Raises every resource cap | No |
| Herb garden | 20 | 0 | 1.8 | — | +1 herb / 14 s | Yes |
| Apothecary | 30 | 20 | 2.0 | — | 2 herbs → 1 medicine / 18 s | Yes |
| Tea house | 30 | 10 | 2.1 | 5.2 | Companions socialise farther | No |
| Memorial | 25 | 10 | 2.2 | 3.2 | Holds memorials for the dead | No |
| Reflection | 40 | 20 | 2.0 | — | Confines a dissenter until loyalty returns | No |
| Sun bath | 25 | 15 | 2.2 | 4.5 | Nearby companions rest and recover loyalty | No |
| Obelisk | 15 | 30 | 2.6 | 5.5 | +1 grace / 12 s and a wide loyalty aura | No |
| Temple | 50 | 30 | 2.8 | 6.0 | +2 grace / 12 s and a loyalty aura | No |
| Sawmill | 35 | 10 | 2.7 | — | +2 wood / 9 s | Yes |
| Stoneworks | 30 | 20 | 2.3 | — | +2 stone / 10 s | Yes |
| Wheat field | 25 | 5 | 1.9 | — | +2 food / 10 s | Yes |
| Bell tower | 40 | 35 | 2.9 | 7.0 | +1 grace / 15 s and the widest aura | No |
| Reliquary | 30 | 25 | 2.1 | 3.5 | +1 pearl / 18 s and a small loyalty aura | No |

The label you see is localised, so the herb garden is "Herb Terrace", the farm
is "Rose Garden" and the temple is the "Grand Temple" in English. `Sanctum.Label`
looks up `building.<type>` in the lexicon and falls back to the raw enum name.

## Production, in detail

Production is timer-based and checked in `Sanctum.Update`. Each cycle adds an
amount to the building's internal `Stored` buffer, which you collect by pressing
++r++ on the building. The buffer's capacity is:

```csharp
int capacity = Capacity + bonus * 4 + levelBonus * 4;
```

`Capacity` is 10, `bonus` is the worker's `WorkBonus` and `levelBonus` is
`Level - 1`. A level-3 building with a skilled worker therefore buffers 22 units
before it stops producing — enough to leave running while you are in the Fallow.

| Sanctum | Interval | Base yield | Per worker point | Per level |
|---|---|---|---|---|
| Farm | 12 s | 2 food | +1 | +1 |
| Lumberyard | 10 s | 1 wood | +1 | +1 |
| Quarry | 12 s | 1 stone | +1 | +1 |
| Herb garden | 14 s | 1 herb | +1 | +1 |
| Sawmill | 9 s | 2 wood | +1 | +1 |
| Stoneworks | 10 s | 2 stone | +1 | +1 |
| Wheat field | 10 s | 2 food | +1 | +1 |
| Shrine | 8 s | 1 grace | +1 | +1 |
| Kitchen | 16 s | 2 food → 1 meal | — | — |
| Apothecary | 18 s | 2 herbs → 1 medicine | — | — |
| Obelisk | 12 s | 1 grace | — | — |
| Temple | 12 s | 2 grace | — | — |
| Bell tower | 15 s | 1 grace | — | — |
| Reliquary | 18 s | 1 pearl | — | — |

Shrines and the four grace buildings are the exception to the `Stored` model:
they credit grace directly and pop a floating "+1 grace" over the roof, so there
is nothing to collect. Farms and wheat fields add `HarvestState.BonusFood` at
collection time, which is how the Harvest creed and the Wheel omen turn a 2-food
cycle into a 3-food cycle.

!!! tip "Workers are multipliers, not requirements"
    Only the kitchen and apothecary refuse to run without a worker. Every other
    producer keeps ticking at its base rate, so an unstaffed farm is slow, not
    dead. Assigning a worker to a farm is usually a better return than building
    a second farm, because the worker adds to yield *and* capacity.

## Condition and dirt

Sanctums wear out. `Update` subtracts `0.04` condition per second from every
building, which is 100 condition in roughly 42 minutes of real time. Condition
scales output linearly from 0.55× at zero to 1× at full:

```csharp
float conditionScale = Mathf.Lerp(0.55f, 1f, Condition / 100f);
```

Nothing stops producing at zero condition; it simply produces at 55%. Repair is
priced from the missing condition, not the building's cost:

| Cost | Formula |
|---|---|
| Wood | `max(1, ceil((100 − condition) × 0.12))` |
| Stone | `max(1, ceil((100 − condition) × 0.08))` |

A building at 50% costs 6 wood and 4 stone; a building at 12% costs 11 wood and
8 stone. Repair always restores to 100, never partially, and the menu refuses
when condition is already above 99.5.

Shrines have a second axis. They accumulate dirt at `0.25` per second, and grace
output is multiplied by `Mathf.Lerp(1f, 0.35f, Dirt / 100f)` — a fully filthy
shrine produces about a third of its listed grace. Companions on cleaning duty
scrub 12 dirt per second, and they will pick a shrine automatically when it is
the dirtiest thing in the hold.

## Managing a sanctum

Pressing ++r++ at a building opens a six-entry menu. The header shows its label,
level and condition.

| Entry | Cost | Result |
|---|---|---|
| Move | Free | Re-enters placement with the building hidden |
| Upgrade | `round(cost × 0.6) + 10` wood, `round(cost × 0.6) + 8` stone | +1 level, max 3 |
| Remove | Refunds half the base wood and stone | Deletes the sanctum and unassigns staff |
| Collect / Craft / Stores / Release | Free | Context action for that type |
| Repair | See above | Restores condition to 100 |
| Enter interior | Free | Switches to the 7 × 5 interior grid |

Upgrading is intentionally cheaper than rebuilding: a Farm upgrade costs 22 wood
and 8 stone, while a second Farm costs 20 wood and occupies new ground. The
trade is space against throughput, and it is the main reason vertical
specialisation beats sprawl in a mature hold.

Removing a building is not destructive to companions. `RemoveSanctum` releases
any penitent, clears the worker's job, unassigns the sleeper, refunds half the
cost, and then re-runs worker and bed assignment for the whole roster.

## Workers, sleepers and hauling

`TryAssignWorker` walks the companion list and gives an unemployed, able
companion the first job that needs one. The `NeedsWorker` set is farm,
lumberyard, quarry, shrine, kitchen, herb garden, apothecary, sawmill, stoneworks
and wheat field. A companion cannot work while sick, dissenting or confined.

Production is not the only labour. A worker whose buffer reaches four units
(two for an herb garden) picks the load up and carries it to the hold's deposit
point:

```csharp
if (Job != null && Job.Stored >= (Job.Type == SanctumType.HerbGarden ? 2 : 4))
    StartHaul();
```

`HaulSpot` is the storage building's front edge if one exists, otherwise the
altar. On arrival, `DepositHaul` credits the resources and the worker walks back.
`FlushHaul` deposits anything in hand before a save or a forced task change, so
resources are never silently lost when you reassign someone mid-carry.

## Storage and caps

Storage is the only building whose effect is global. Each level adds to every
cap, and `HarvestState.NotifyChanged` clamps all eight resources after every
change, so overflow is impossible.

| Resource | Base cap | Per storage level |
|---|---|---|
| Wood | 250 | +100 |
| Stone | 250 | +100 |
| Food | 100 | +50 |
| Herbs | 30 | +20 |
| Meals | 30 | +20 |
| Medicine | 10 | +5 |
| Bombs | 5 | +2 |
| Potions | 20 | +0 |

Bombs also gain +2 from the Magician omen and +2 from the Lantern heirloom;
potions gain +5 from the Hermit omen. Those bonuses apply whether or not you own
a storage, but the storage multiplier is what makes a bomb-heavy run practical.

## Bell tower, temple and the loyalty auras

Auras are checked once per frame per companion in `FollowerAgent`. They are
small, continuous drips rather than bursts, and they stack only in the sense
that a companion standing in two auras gets both.

| Building | Per-second effect inside the aura |
|---|---|
| Shrine | +0.018 loyalty |
| Tea house | +0.01 loyalty |
| Sun bath | +0.01 loyalty, −0.08 exhaustion |
| Obelisk | Loyalty aura, plus grace production |
| Temple | Loyalty aura, plus grace production |
| Bell tower | Loyalty aura, plus grace production |
| Reliquary | Small loyalty aura, plus pearls |

The Tea House is the social building: its 5.2 m radius lets companions find
conversation partners farther apart, which is what drives friendship and romance
checks. The Sun Bath is the recovery building: it burns exhaustion while adding
loyalty, making it the pair to a workshop district.

## Interiors

Choosing **Enter interior** hides the hub and builds a small room around you.
The floor is 8.2 m across, movement is bounded to ±3.6 by ±3.1, and the camera
pulls in to 8.2. The buildable grid is 7 columns by 5 rows:

```csharp
static Vector2Int InteriorCell(Vector3 position) =>
    new Vector2Int(
        Mathf.Clamp(Mathf.RoundToInt(position.x), -3, 3),
        Mathf.Clamp(Mathf.RoundToInt(position.z), -2, 2));
```

Pieces are placed on the cell you are standing on, and an occupied cell is
refused. Costs are deliberately small:

| Piece | Cost | Notes |
|---|---|---|
| Wall | 5 wood | Solid; blocks movement |
| Door | 10 wood | Walk-through, the exit prompt |
| Candle | 3 pearls | Decoration; +0.6 aura to the parent sanctum |

Each candle permanently raises `InteriorAuraBonus` on the sanctum by 0.6 m, and
removing one lowers it again. That is the only way to grow an aura beyond its
base radius, and it applies to whichever aura the building already has — a
bell tower with three candles projects 8.8 m instead of 7.

!!! note "Interiors are saved, not rebuilt from a template"
    Every piece is written into `SanctumData.interior` as a `type, x, z` triple.
    `BuildInterior` replays that list on load. Removing a piece removes its entry
    and its aura bonus, so a save always matches what you see.

## Under the hood

`Sanctum` (`Scripts/Buildings/Building.cs`) is a plain MonoBehaviour with no
ScriptableObject backing. The type, level, stored amount, dirt, condition and
three companion references are public fields; costs and aura radii are static
switch expressions on the type.

`HarvestGame` owns the list of placed sanctums and mirrors it into
`HarvestState.Data.buildings` as `SanctumData` records for saving. The two lists
are index-aligned, which is why moving or removing a building has to update both
in the same method — see `ConfirmPlacement` and `RemoveSanctum`.

Placement validity is deliberately geometric rather than physics-based:

```csharp
bool geometry = IsPlacementValid(placementType, placementCursor, placementMoving);
bool affordable = placementMoving != null ||
                  (HarvestState.Data.wood >= Sanctum.CostWood(placementType) &&
                   HarvestState.Data.stone >= Sanctum.CostStone(placementType));
```

There is no overlap test against gathering nodes or companions inside
`IsPlacementValid` itself; those checks run earlier in the same method, before
the distance loop, so keep them together if you extend the rule set.

`RebuildNavGrid` runs after every placement, move and removal. Companions use a
coarse grid rather than a navmesh, so a wall placed across the middle of the
hold can genuinely trap someone until the next rebuild — which is exactly what
happens, because the rebuild is triggered by the placement, not by the trapped
companion.

## Related

- [The Harvesthold hub and companions](hub-and-companions.md) — the roster that staffs these buildings.
- [Economy and production](economy.md) — caps, meals, crafting and merchant prices.
- [Ceremonies, omens and heirlooms](ceremonies.md) — the buffs that multiply output.
- [Combat in the Fallow](combat.md) — what the apothecary's bombs are for.
