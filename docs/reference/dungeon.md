# Dungeon API reference

`Expedition` builds and runs a pilgrimage floor one room at a time: it generates a
room graph, dresses a single room, spawns husks, traps and the act's Warden, then
opens the doors when the room is cleared. Everything below lives in the single
`ChosenOfTheHarvest` namespace and is created at runtime — no prefabs to wire.

!!! warning "File names and class names differ"
    The files are named after the feature, not the type they contain. Search for the
    class: `Dungeon.cs` holds `Expedition`, `Enemy.cs` holds `Creature`,
    `BossController.cs` holds `WardenController`, `BossArena.cs` holds `WardenArena`,
    `DungeonTrap.cs` holds `ExpeditionTrap`, `PressurePlate.cs` holds `RunePlate`,
    `Projectile.cs` holds `HexBolt`, and `FloorPlan.cs` holds `FloorPlanGenerator`.

## At a glance

`Expedition` owns the floor and builds rooms; `FloorPlanGenerator` decides the graph;
`Creature` and `WardenController` fight; `WardenArena` dresses the boss room;
`ExpeditionTrap`, `RunePlate` and `HexBolt` are the small `MonoBehaviour` props; and
`RoomCollision` provides XZ bounds and pillar collision without Unity physics.

## Expedition

The runtime owner of a pilgrimage. `HarvestGame` creates it, calls `Begin`, and drives
it through `Prompt`/`Interact` when the sovereign presses interact.

**Identity and progression**

| Member | Type | Description |
|---|---|---|
| `ActIndex` | `int` | Act 0–2, clamped by `Begin` |
| `Floor` | `int` | Current floor, starts at 1 |
| `FloorCount` | `int` | Floors per act; `Begin` sets it to 2 |
| `HasNextFloor` | `bool` | `Floor < FloorCount` |
| `CurrentBiome` | `Biome` | Cast from `ActIndex` |
| `BiomeName` | `string` | Ashen Orchard / Drowned Conservatory / Ember Citadel |
| `WardenName` | `string` | Act warden, plus `" Ascendant"` when `Floor > 1` |
| `Finished` | `bool` | Set when the run ends |
| `Rooms` | `IReadOnlyList<FloorRoom>` | The floor graph |
| `RoomIndex`, `RoomCount` | `int` | Current node and node count |

**Current room**

| Member | Type | Description |
|---|---|---|
| `CurrentRoomType` | `RoomType` | Type of the current node |
| `RoomTypeLabel` | `string` | Localised room name, `"Corridor"` while in a corridor |
| `ObjectiveText` | `string` | Localised objective for the current room state |
| `Layout` | `LayoutKind` | Open, Pillars, Colonnade or Cross |
| `RoomHalfWidth`, `RoomHalfDepth` | `float` | Room extents in world units |
| `ObstacleCount` | `int` | `RoomCollision.Count` |
| `PlayerStart`, `DoorSpot`, `ChestSpot`, `ExitSpot` | `Vector3` | Generated anchors (`DoorPosition` aliases `DoorSpot`) |

**Progress flags**

| Member | Type | Description |
|---|---|---|
| `DoorReady`, `ChestReady` | `bool` | Room doors / chest are interactable |
| `InCorridor` | `bool` | Between rooms; only the far door is active |
| `Keys` | `int` | Keys carried through this floor |
| `LockedEdgeCount` | `int` | Locked edges in the floor graph |
| `LockedDoorCount` | `int` | Locked doors still standing in the room |
| `RunesActivated`, `RuneTotal` | `int` | Puzzle progress |
| `TrapCount` | `int` | Traps spawned in this room |
| `CreatureCount` | `int` | `Creature.All.Count` |

**Methods**

| Member | Returns | Description |
|---|---|---|
| `Begin(HarvestGame owner)` | `void` | Starts an act derived from `pilgrimages`, minus one |
| `Begin(HarvestGame owner, int act)` | `void` | Starts a specific act 0–2 |
| `End()` | `void` | Despawns creatures, destroys the room, clears collision |
| `BuildRoom()` | `void` | Rebuilds the current node (resets all room flags) |
| `TravelTo(int id)` | `void` | Enters a node with the transition flash |
| `UseDoorAt(int index)` | `bool` | Interacts with a door by index |
| `FinishCorridor()` | `void` | Enters the corridor's target room |
| `Prompt(Vector3 playerPosition)` | `string` | Interaction prompt for the player's position |
| `Interact(Vector3 playerPosition)` | `bool` | Runs the interaction; `true` if something happened |
| `OpenDoors()` | `void` | Marks the room cleared and enables doors |
| `ActivateRune(RunePlate plate)` | `void` | Puzzle callback; opens doors when all runes are lit |
| `OpenChest()` | `void` | Pays pearls/wood/stone, plus a key in treasure rooms |
| `ExitRun()` | `void` | Descends to the next floor or exits the expedition |
| `SpawnSummon(Vector3 center)` | `void` | Summoner hook; refuses at 18 live creatures |
| `SpawnTestCreature(Archetype kind, bool elite)` | `Creature` | Smoke-test hook; `null` outside a built room |

!!! note "Hard-coded warden and biome names"
    `WardenName` and `BiomeName` return literal strings rather than `Lexicon`
    lookups, so renaming a warden means editing `Expedition` as well as the
    localisation tables.

## FloorPlanGenerator

Builds a fully connected grid, promotes one room per role, then locks a few side
edges.

| Member | Returns | Description |
|---|---|---|
| `Generate(int act, out HashSet<int> lockedEdges)` | `List<FloorRoom>` | Room list in id order; id 0 is the entrance, the last id is the Warden |
| `EdgeKey(int a, int b)` | `int` | Stable key: `min * 100 + max` |
| `Name(RoomType type)` | `string` | English fallback through `Lexicon.Get("room.<type>", …)` |

The grid is `5 × 2` for act 0, `8 × 2` for act 1 and `5 × 4` for act 2 — 10, 16 and
20 rooms per floor, 92 rooms across three acts with two floors each. Every floor
promotes shop, treasure, curse, puzzle, rescue and forge once, adds `1 + act` elite
rooms (plus a second treasure room from act 2), and locks `2 + act` edges; rooms 0
and the Warden room are never lock endpoints.

**RoomType and its localised label**

| `RoomType` | Label | Key |
|---|---|---|
| `Entrance` | Entrance | `room.entrance` |
| `Combat` | Combat | `room.combat` |
| `Elite` | Elite | `room.elite` |
| `Treasure` | Treasure | `room.treasure` |
| `Shop` | Shop | `room.shop` |
| `Curse` | Curse | `room.curse` |
| `Puzzle` | Rune Puzzle | `room.puzzle` |
| `Rescue` | Rescue | `room.rescue` |
| `Forge` | Forge | `room.forge` |
| `Warden` | Warden | falls back to `"Warden"` |

!!! warning "`room.warden` is missing from `en.json`"
    `Name()` asks for `"room." + type.ToString().ToLowerInvariant()` — that is
    `room.warden`, while the table defines `room.boss`. The Warden room therefore
    shows the English fallback. Add a `room.warden` entry, or point the fallback at
    `room.boss`, before shipping a translation.

**FloorRoom** is a plain class, not a `MonoBehaviour`:

| Field | Type | Description |
|---|---|---|
| `Id` | `int` | Index in the room list |
| `Grid` | `Vector2Int` | Column/row on the plan |
| `Type` | `RoomType` | Role assigned by the generator |
| `Visited`, `Cleared` | `bool` | Minimap and door state |
| `Neighbors` | `List<int>` | Connected room ids |

## RoomCollision

Static XZ collision. Obstacles are vertical cylinders; the room is a clamped box,
so movement is deterministic and needs no `Rigidbody`.

| Member | Returns | Description |
|---|---|---|
| `HalfBounds` | `Vector2` | Half extents of the walkable box |
| `Enabled` | `bool` | `false` until `Reset` is called |
| `Count`, `Obstacles` | `int`, `IReadOnlyList<Obstacle>` | Registered pillars |
| `Reset(Vector2 halfBounds)` | `void` | Clears obstacles, sets bounds, enables |
| `Clear()` | `void` | Drops obstacles and disables collision |
| `Add(Vector3 center, float radius)` | `void` | Registers a cylinder (Y is ignored) |
| `IsFree(Vector3 position, float radius)` | `bool` | Circle-vs-bounds and circle-vs-cylinder test |
| `Resolve(Vector3 position, float radius)` | `Vector3` | Clamps to bounds, pushes out of obstacles (3 passes) |
| `DistanceToSegment(Vector3 point, Vector3 a, Vector3 b)` | `float` | XZ distance, used to keep spawns off the entry path |

`Obstacle` exposes `Center` and `Radius`. `Resolve` is what creatures, bolts and
companions call every frame instead of a physics query.

## ExpeditionTrap

A glowing circle that telegraphs for 0.75 s, then damages anything inside.
`Spawn(Transform parent, Transform playerTarget, Vector3 ground, float trapRadius,
int trapDamage)` creates an inactive trap with a random 1.2–2.2 s first cooldown.

Damage lands when the player is within `radius + 0.25` at the end of the telegraph,
then the cooldown rerolls to 1.6–2.6 s. Combat rooms spawn 2 traps and elite rooms
3, with damage `8 + pilgrimages` and radius 0.95–1.25 — see `Expedition.SpawnTraps`.

## RunePlate

One plate of the three-rune puzzle. `Spawn(Expedition owner, Transform parent,
Vector3 ground, int index)` creates it and pulses it until the sovereign comes
within 0.8 units, then it turns green, plays `pickup`, and calls
`owner.ActivateRune(this)` exactly once.

## HexBolt

Pooled enemy projectile — a camera-facing sprite that travels on the ground plane.
`Spawn(Vector3 position, Vector3 direction, int damage, Color tint, float speed =
5.5f)` reuses an inactive bolt, or creates one while the pool is under 48.

Bolts live 4 s, despawn on room geometry (`RoomCollision.IsFree` at radius 0.12),
and hit the sovereign within 0.6 units for `damage`. `Creature.CastAtPlayer` and
`WardenController` fan/radial attacks are the only callers.

## Creature
A husk on the XZ plane. Brains differ; the visual pose is a `PoseAnimator`.

| Member | Type | Description |
|---|---|---|
| `All` | `static List<Creature>` | Every live husk; room clear checks read `Count` |
| `MaxHealth`, `Health`, `Damage` | `int` | Current combat numbers |
| `Speed`, `AttackRange`, `AttackCooldown` | `float` | Set by `Init` from the archetype |
| `Kind` | `Archetype` | Behaviour brain |
| `IsWarden` | `bool` | Boss flag; suppresses exploding and swaps the brain |
| `Elite` | `EliteModifier` | Applied by `ApplyElite` |
| `Phase` | `int` | 1 or 2; only `EnterPhaseTwo` raises it |
| `DamageMultiplier`, `SpeedMultiplier` | `float` | Curse and act scalars |
| `DamageReduction` | `float` | 0–1, subtracted in `Hit` |
| `MovementLocked`, `AttacksEnabled` | `bool` | Set by telegraphed routines |
| `BoltDamage` | `int` | `round(Damage * 0.8 * DamageMultiplier)`, minimum 1 |

| Method | Description |
|---|---|
| `Init(Transform playerTarget, Color tint, int health, int damage, float scale, bool boss, Archetype kind = Archetype.Melee)` | Sets health (times `Preferences.CreatureHealthMultiplier`), damage, archetype stats and timers |
| `Hit(int damage)` | Applies reduction, floating text, sound, and `Die` at 0 |
| `Heal(int amount)` | Clamps to `MaxHealth`, shows green text |
| `Knockback(Vector3 direction, float force)` | Adds to a decaying knock velocity |
| `ApplyBleed/ApplyBurn/ApplyPoison(float duration, float dps)` | Refreshes a damage-over-time effect |
| `ApplySlow(float factor, float duration)` | Factor clamped to 0.25–1 |
| `ApplyElite(EliteModifier modifier)` | Records and applies the modifier |
| `CastAtPlayer()` | Fires a `HexBolt`; timer 1.7 s for archers, 2.5 s otherwise |
| `PlayTelegraphAnimation(float duration)` | Plays the tell pose |
| `EnterPhaseTwo()` | +12 % scale, +15 % speed, ×0.8 cooldown, red tint |
| `Die()` | Pays `1 + HarvestState.BonusPearls` pearls, then destroys after 0.32 s |
| `Despawn()` | Removes without loot (room rebuild, expedition exit) |

Damage-over-time ticks are fixed: bleed 0.5 s, burn 0.45 s, poison 0.55 s, each for
`ceil(dps × interval)`.

### Archetype

| Archetype | Speed | Range | Cooldown | Behaviour |
|---|---|---|---|---|
| `Melee` | 2.3 | 1.05 | 1.1 | Chase, 0.34 s tell, hit within 1.55× range |
| `Fast` | 3.15 | 0.95 | 0.85 | Melee brain, fastest grunt |
| `Brute` | 1.65 | 1.35 | 1.45 | Melee brain, long reach, heavy hit |
| `Caster` | 2.0 | 1.05 | 1.3 | Holds 3.1–6.4 units, fires bolts on a 2.5 s timer |
| `Charger` | 2.5 | 1.0 | 1.2 | 0.45 s windup, 0.28 s dash at 13 u/s, then 2.8–4.2 s |
| `Exploder` | 2.9 | 0.9 | 1.0 | Melee brain; explodes on death for `max(6, Damage * 1.4)` in 2.4 units |
| `Shielder` | 1.5 | 1.2 | 1.7 | Melee brain with 20 % damage reduction |
| `Summoner` | 2.1 | 1.05 | 1.6 | Kites under 4.2, approaches past 7, summons 2 `Fast` every 8–10 s |
| `Healer` | 2.3 | 1.05 | 1.5 | Holds 4–8 units, heals allies within 5.5 for `max(8, 16 % max health)` |
| `Archer` | 2.5 | 1.05 | 1.5 | Holds 4.5–8.5 units, fires from beyond 2.2 on a 1.7 s timer |
| `Slammer` | 1.6 | 1.7 | 2.1 | At 2.8 units starts a 0.55 s ground slam for 1.2× damage |
| `Assassin` | 3.35 | 1.0 | 0.9 | Blinks beside the sovereign when further than 2.4 units (0.35 s tell) |

### Spawn budget

`Expedition.SpawnCreature` derives everything from the archetype. With
`d = HarvestState.Data.pilgrimages` and `f = Floor - 1`:

| Archetype | Base health | Base damage | Visual scale |
|---|---|---|---|
| `Melee` | `55 + 5d + 20f` | `10 + d/2 + f` | 0.55 |
| `Fast` | `35 + 4d + 15f` | `7 + d/3 + f` | 0.42 |
| `Brute` | `120 + 8d + 30f` | `15 + d + 2f` | 0.80 |
| `Caster` | `45 + 4d + 15f` | `8 + d/3 + f` | 0.50 |
| `Charger` | `62 + 5d + 18f` | `12 + d/2 + f` | 0.58 |
| `Exploder` | `30 + 3d + 12f` | `9 + d/3 + f` | 0.48 |
| `Shielder` | `110 + 6d + 25f` | `11 + d/2 + f` | 0.72 |
| `Summoner` | `55 + 4d + 16f` | `7 + d/3 + f` | 0.55 |
| `Healer` | `60 + 4d + 16f` | `6 + d/3 + f` | 0.52 |
| `Archer` | `40 + 3d + 13f` | `9 + d/3 + f` | 0.48 |
| `Slammer` | `140 + 8d + 30f` | `16 + d + f` | 0.88 |
| `Assassin` | `42 + 3d + 14f` | `11 + d/2 + f` | 0.44 |
| Warden | `180 + 12d + 60·act + 90f` | `18 + d + 3·act + 4f` | `1 + 0.08·act` |

Combat rooms spawn 4 husks, elite rooms 5, and the archetype is picked
deterministically as `kinds[(i + ActIndex + d) % 12]`, so a room is a slice of the
archetype wheel rather than a random handful.

## Elite modifiers

`EliteModifier` has `None` plus the three below. Elites are picked as
`(index + ActIndex) % 3 + 1` and already carry ×1.5 health, ×1.3 damage, ×1.08 scale
and a 35 % blend towards amber before the modifier lands.

| Modifier | Effect | Verified in |
|---|---|---|
| `Armored` | `DamageReduction` raised to at least 0.3 (30 % less damage taken) | `ApplyElite` |
| `Frenzied` | `SpeedMultiplier *= 1.25`, `AttackCooldown *= 0.8` | `ApplyElite` |
| `Vampiric` | Melee hits heal the husk for 30 % of its `Damage` | `MeleeRoutine` |

## WardenController

Layered on top of the boss `Creature`; the creature still walks, takes damage and
dies through the normal path.

| Member | Type | Description |
|---|---|---|
| `Phase` | `int` | 1 or 2 |
| `IsTelegraphing` | `bool` | `true` while a routine owns the boss |
| `AttackTimer` | `float` | Seconds until the next attack |
| `Init(Creature owner, Transform playerTarget, Transform effectsParent, int actIndex = 0)` | `void` | Wires the boss; clamps the act |
| `MovesetSummary(int act)` | `static string` | One-line description per act |

Phase two begins automatically at 50 % health. Routines lock movement, telegraph,
then deal damage and release:

| Routine | Trigger | Effect |
|---|---|---|
| Slam | Default attack | Radius 2.8 (phase 1) / 3.5, damage 18 / 24; act riders below |
| Antler charge / volley (act I) | Every 4th attack | 0.62 s windup, 0.3 s dash at 17 u/s, 16 / 21 damage; phase 2 volley is a 9-bolt fan plus a 6-bolt ring |
| Thorn snare (act II) | Every 3rd attack | Three roots near the player, 12 / 17 damage within 1.35; phase 2 adds a 5-bolt fan |
| Mirage surge (act III) | Every 3rd attack | 0.68 s expanding ring, 15 / 21 damage within 4.8, then teleport and a radial volley |
| Slam riders | Per act | Act II adds 6/11 bolts and a phase-2 fan; act III adds 8/14 bolts and a phase-2 teleport; act I adds 8 bolts only in phase 2 |

## WardenArena

Static per-act dressing, built before the boss spawns: `Theme(int act)` returns
Gilded Court / Drowned Ring / Obsidian Court, and `Build(Transform parent, int act,
float halfWidth, float halfDepth)` adds the pillars, reeds or tide pools for that act.

## Warden roster

Three Wardens, one per act, two floors each. The second floor is the Ascendant
form — the same name with `" Ascendant"` appended and tougher stats.

| Act | Biome | Arena | Warden (floor 1 / floor 2) | Moveset |
|---|---|---|---|---|
| I | Ashen Orchard | Gilded Court | The Gilded Stag / The Gilded Stag Ascendant | antler charge + antler volley + slam |
| II | Drowned Conservatory | Drowned Ring | The Mire Queen / The Mire Queen Ascendant | thorn snare + mire fan + slam |
| III | Ember Citadel | Obsidian Court | The Obsidian Sultan / The Obsidian Sultan Ascendant | mirage surge + teleport + radial bolts |

Boss art is one generated body sprite per act (`Art/Wardens/stag_body`,
`mire_body`, `sultan_body`) attached by `WardenAppearance.Init`; the death lines are
"the antlers fall", "the mire recedes" and "the obsidian cracks".

## Encounter composition

The Warden room spawns exactly three creatures: the boss (`IsWarden`) plus a `Brute`
and a `Caster` escort. Clearing it sets `ChestReady` and plays the death routine;
only on the final floor does it report the act to `HarvestGame`. Objective text
switches from `dungeon.slay` to `dungeon.bosschest`.

```csharp
// What is the sovereign looking at right now?
var run = HarvestGame.Instance.ExpeditionRef;
Debug.Log($"{run.BiomeName} {run.Floor}/{run.FloorCount} - {run.RoomTypeLabel}");
Debug.Log(run.ObjectiveText);

// QA: put one archetype (optionally elite) in the current room, then burn it.
run.SpawnTestCreature(Creature.Archetype.Slammer, elite: true);
foreach (var husk in Creature.All) husk.ApplyBurn(3f, 6f);
```

## Related

- [Followers API](followers.md) — the hub simulation that feeds expeditions
- [Expeditions](../guide/expeditions.md) — player-facing flow and room types
- [Combat & Wardens](../guide/combat.md) — weapons, rites and boss strategy
- [Balancing & tuning](../customization/balancing.md) — where the numbers live
