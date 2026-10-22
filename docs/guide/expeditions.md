# Expeditions into the Fallow

The Fallow is the template's roguelike half. It is entered through the portal in
the Harvesthold, it is generated one room at a time from a fixed act budget, and
it is the only place where pearls, omens and heirlooms come from. This page
covers the curse roll, the floor graph, every room type, keys and corridors, the
act pacing, and how a run ends.

!!! warning "A run is a snapshot"
    The Fallow is not a separate scene. `EnterExpedition` disables `hubRoot`,
    resets your health, rolls a curse, and hands control to `Expedition.Begin`.
    When you leave, `ExitExpedition` re-enables the hub at the exact position you
    left from — building timers and companion needs resume where they stopped.

## Entering the portal

Standing at the portal and pressing ++e++ opens `ShowCampaignMap()`, not the
dungeon. The map lists three acts plus **Devotion upgrades** and **Quest log**.
An act that is ahead of `campaignStage` is refused with "Clear the previous
warden first". Choosing an available act shows an intro card, and only its
**Begin pilgrimage** entry calls:

```csharp
public void EnterExpedition(int act)
{
    if (mode != Mode.Hub || ceremonyRunning) return;

    currentAct = Mathf.Clamp(act, 0, 2);
    HarvestState.RollCurse();
    mode = Mode.Expedition;
    hubReturnPosition = Player.transform.position;
    hubRoot.SetActive(false);

    Audio.Play("portal", 0.8f);
    Audio.PlayMusic("music_dungeon");
    ...
    HarvestState.Data.pilgrimages++;
    Player.SetHealth(Player.MaxHealth);
    ExpeditionRef.Begin(this, currentAct);
}
```

`pilgrimages` is not cosmetic: it is the depth scalar used by chest pearls, trap
damage, and every husk's health and damage formula.

## Curses rolled on entry

`HarvestState.RollCurse()` picks uniformly from the three real curses — it can
never return `Curse.None`:

| Curse | Effect | Where it is applied |
|---|---|---|
| Fury | +30% enemy damage | `Creature.DamageMultiplier` at spawn |
| Frailty | −25 maximum health | `RunHealthPenalty`, subtracted in `MaxHealth` |
| Gloom | +25% enemy speed | `Creature.SpeedMultiplier` at spawn |

The curse is runtime-only (`HarvestState.CurrentCurse`) and is cleared by
`ClearRunState()` on exit, so it never leaks into a save. It is printed in the
HUD status line next to the husk count.

## Act structure

Three acts, two floors each, six floors in total. Each floor is regenerated from
scratch, so the graph is different every descent.

| Act | Biome | Warden | Floor grid | Rooms per floor | Rooms per act |
|---|---|---|---|---|---|
| 0 | Ashen Orchard | The Gilded Stag | 5 × 2 | 10 | 20 |
| 1 | Drowned Conservatory | The Mire Queen | 8 × 2 | 16 | 32 |
| 2 | Ember Citadel | The Obsidian Sultan | 5 × 4 | 20 | 40 |

`JourneyPacing` (`Scripts/Core/CampaignPacing.cs`) states the same budget as
constants and adds the intended session length:

| Constant | Value |
|---|---|
| `ActOneRooms` | 20 |
| `ActTwoRooms` | 32 |
| `ActThreeRooms` | 40 |
| `TotalRooms` | 92 |
| `TargetMinutesMin` / `Max` | 80 / 120 |
| `Summary` | `92 rooms • 80-120 min target` |

The warden reward budget is 8 / 12 / 16 devotion, which `WardenDefeated` computes
as `8 + act * 4` before the `crown` heirloom multiplier.

## How a floor is generated

`FloorPlanGenerator.Generate(act, out lockedEdges)` builds the whole graph before
the first room is drawn. It lays out a rectangular grid, connects every room to
its right and lower neighbour (so every room is reachable), marks `rooms[0]` as
the entrance and the last room as the warden, then distributes role rooms into
the middle.

```csharp
int columns = act switch { 1 => 8, 2 => 5, _ => 5 };
int rows = act == 2 ? 4 : 2;
int bossId = columns * rows - 1;
...
var roles = new List<RoomType>
{
    RoomType.Shop, RoomType.Treasure, RoomType.Curse,
    RoomType.Puzzle, RoomType.Rescue, RoomType.Forge
};
int elites = 1 + act;
for (int i = 0; i < elites; i++) roles.Add(RoomType.Elite);
if (act >= 1) roles.Add(RoomType.Treasure);
```

The role list is shuffled, then written into consecutive room ids from 1 upwards;
anything left before the warden becomes a plain `Combat` room. That guarantees
every utility room appears at least once per floor, with one elite in act one,
two in act two and three in act three, plus a second treasure room from act two.

Finally it locks `Mathf.Min(2 + act, candidates.Count)` edges between ordinary
rooms — never an edge touching the entrance or the warden — and returns them as
packed `EdgeKey(a, b)` integers. Those become the locked doors.

## Room types

`RoomType` has ten members. `FloorPlanGenerator.Name` localises them through the
`room.*` lexicon keys.

| Room | Objective | What is inside |
|---|---|---|
| `Entrance` | none | Spawn room; doors open immediately |
| `Combat` | clear the room | 4 husks, doors open at zero |
| `Elite` | defeat the elite | 5 husks, one carries an elite modifier, 3 traps |
| `Treasure` | open the chest | Chest gives pearls, wood, stone **and a key** |
| `Shop` | trade, then choose a door | Merchant with limited stock and a sell menu |
| `Curse` | touch the altar | Take the gift: +1 key, +25 pearls, −20 health |
| `Puzzle` | step on 3 runes | Three `RunePlate` triggers; all three open the doors |
| `Rescue` | free the captive | `RecruitCompanion()`, blocked at the companion cap |
| `Forge` | use the forge | Sharpen blade for 15 pearls (+5 damage) |
| `Warden` | slay the warden | Boss + two escorts, then a boss chest |

`Expedition.Update` watches `Creature.All.Count` for combat and elite rooms and
calls `OpenDoors()` the moment the room is empty. There is no wave logic — a
summoner that keeps adding minions keeps the room open.

!!! note "Temperance"
    The `temperance` omen heals 8 health on every cleared room **except** the
    entrance, and the toast changes to say so. It is one of the strongest
    sustain picks in the deck; see [Ceremonies and omens](ceremonies.md).

## Keys, doors and corridors

Keys are a per-floor counter, reset to 0 on every descent. There are three
sources: a treasure chest (+1), the merchant (10 pearls), and the cursed altar
(+1). Each locked door costs exactly one key and stays unlocked afterwards.

Doors behave in a strict order:

1. If the room is not cleared, the door refuses and prints the objective.
2. If the door is locked and you have a key, it consumes the key, recolours
   itself lilac and stays in place — you must press again.
3. Otherwise it builds a corridor and moves you into the target room.

A corridor is its own tiny room: `RoomHalfWidth = 3.2`,
`RoomHalfDepth = 5.5`, no enemies, no chest, one door at the far end. The label
above that door names the room type you are about to enter, which is the only
preview the game gives you before committing.

## The pilgrimage map and campaign stages

`CampaignStage` is `Data.campaignStage` clamped to 0–3. It is the number of
wardens you have broken, and it gates the act list: act 0 is available at stage
0, act 1 at stage 1, act 2 at stage 2. The map also exposes the devotion screen
and the quest log.

`WardenDefeated(act, bossName)` only pays out the **first** time an act's warden
falls:

```csharp
if (act + 1 > HarvestState.Data.campaignStage)
{
    HarvestState.Data.campaignStage = act + 1;
    int reward = Mathf.RoundToInt((8 + act * 4) *
        (HarvestState.HasHeirloom("crown") ? 1.15f : 1f));
    HarvestState.Data.devotion += reward;
    ...
}
```

It is called from `Expedition.Update` only when the warden dies on the **last**
floor (`!HasNextFloor`), so a first-floor warden kill does not advance the
campaign.

## Descending and finishing

A warden room is the last room of a floor. Killing the warden opens a chest; the
chest's omen choice then opens a door labelled `DESCEND` (or `RETURN` on the
final floor). `ExitRun()` decides:

| Condition | Result |
|---|---|
| `HasNextFloor` | `BeginNextFloor()`: floor++, keys reset, new graph, **heal 20% max health** |
| last floor | `Finished = true`, `boot.ExitExpedition(true)` |

`FloorCount` is fixed at 2. The second-floor warden is named with the `Ascendant`
suffix (`WardenName` appends it when `Floor > 1`), gets +90 health and +4 damage
from `floorBonus`, and its arena is the same theme as the first floor.

Leaving with loot autosaves if autosave is enabled, and if `campaignStage >= 3`
and the victory screen has not been seen, `ShowVictory()` runs once and grants
+10 devotion.

## Dying in the Fallow

`OnPlayerDied` in the expedition branch takes half your pearls, exits the run
with `loot = false`, and wakes you at `max(40, MaxHealth / 2)`. There is no
confiscation of wood, stone, heirlooms or omens — pearls are the only death tax.

## Rewards

| Source | Reward |
|---|---|
| Any husk kill | `1 + BonusPearls` pearls (tower omen, ledger heirloom, Greed creed) |
| Chest | `10 + pilgrimages × 2` pearls, +5 wood, +4 stone |
| Treasure chest | the same, plus +1 key, and **no** omen choice |
| Non-treasure chest | the same, then an omen offer of two unowned cards |
| Merchant | heirloom 25 pearls, attire 30 pearls, potion 18, key 10, heal 30 hp for 15 |
| Cursed altar | +1 key, +25 pearls, −20 health |
| Warden (first kill) | 8 / 12 / 16 devotion |
| Every omen already owned | the omen offer converts to +25 pearls |

```csharp
public void OpenChest()
{
    if (!ChestReady) return;
    ...
    int depth = HarvestState.Data.pilgrimages;
    int pearls = 10 + depth * 2;
    HarvestState.Data.pearls += pearls;
    HarvestState.Data.wood += 5;
    HarvestState.Data.stone += 4;
    ...
    if (CurrentRoomType == RoomType.Treasure) { keys++; ... return; }

    boot.ShowOmenMenu(OpenExit);
}
```

## Shop rooms

`ShowShop` builds a six-entry merchant menu with per-room stock that resets on
every room build: 3 heals, 2 keys, 1 heirloom, 2 potions, 1 attire. The heirloom
and attire offers are rolled from `RandomUnowned()` at room build time, so a shop
can never sell you something you already own — and shows "no heirloom to sell"
when the catalogue is exhausted. The sell menu trades 10 wood or stone for 8
pearls, 5 food for 6, and 2 herbs for 5.

## Under the hood

The run starts in `Expedition.Begin`, which clamps the act, fixes the floor count
and generates the first graph:

```csharp
public void Begin(HarvestGame owner, int act)
{
    ActIndex = Mathf.Clamp(act, 0, 2);
    Floor = 1;
    FloorCount = 2;
    BeginRun(owner);
}
```

Only one room exists at a time. `EnterRoom` destroys the previous `roomRoot`,
despawns every `Creature`, resets per-room flags (`shopUsed`, `curseUsed`,
chest/door/exit state), and rebuilds from the room type. `Expedition.End` is the
mirror image: it despawns enemies, destroys the room and clears
`RoomCollision`, which is why leaving mid-room is safe.

!!! tip "Changing the run length"
    `FloorCount = 2` and the three `JourneyPacing` room constants are the only
    places the 6-floor / 92-room budget is expressed. Changing the grid sizes in
    `FloorPlanGenerator.Generate` changes the *actual* rooms per floor, so update
    both if you want the HUD and the pacing summary to agree.

## Related

- [Combat and Wardens](combat.md) — archetypes, elites, traps and the three boss fights
- [Ceremonies and omens](ceremonies.md) — the 18-card omen deck that chests draw from
- [Economy and resources](economy.md) — pearls, heirlooms and the merchant's prices
- [Campaign and quests](quests.md) — the nine-stage chain that tracks these acts
