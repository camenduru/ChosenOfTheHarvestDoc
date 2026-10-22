# Balancing & tuning

Every number that matters is a constant or a catalog entry in C#. This page maps
the tuning surface so you know which file to open.

## Difficulty

Difficulty is a three-step global modifier stored in `Preferences.Difficulty`
(0 = Easy, 1 = Normal, 2 = Hard).

| Difficulty | Incoming damage | Creature health |
|---|---|---|
| Easy | ×0.7 | ×0.85 |
| Normal | ×1.0 | ×1.0 |
| Hard | ×1.3 | ×1.2 |

```csharp
// Scripts/Core/Settings.cs
public static float IncomingDamageMultiplier => Difficulty switch
{
    0 => 0.7f,
    2 => 1.3f,
    _ => 1f
};
```

Add a fourth difficulty by extending both switches and the clamp in
`SetDifficulty`.

## Player power

Player damage is assembled in `HarvestState.WeaponDamage`, which combines the
armament's `BaseDamage`, the sharpening level, omens, creeds, heirlooms and
attire.

| Lever | File |
|---|---|
| Base damage, cooldown, range, combo | `Scripts/Combat/WeaponCatalog.cs` |
| Sharpening step (+5 per purchase) | `GameBootstrap.SharpenBlade` |
| Omen damage bonuses | `GameState.cs` (`WeaponDamage`, `BonusDamage`) |
| Heirloom damage bonuses | `GameState.cs` |
| Attire damage bonuses | `GameState.cs` |
| Critical chance and multiplier | `GameState.cs` |

## Pacing

`Scripts/Core/CampaignPacing.cs` holds the campaign budget:

```csharp
public const int ActOneRooms = 20;
public const int ActTwoRooms = 32;
public const int ActThreeRooms = 40;
public const int TargetMinutesMin = 80;
public const int TargetMinutesMax = 120;
```

These are display/budget constants. The actual generated room count comes from
`FloorPlan`, so change both if you change one.

## Sanctums

| Lever | Location |
|---|---|
| Wood cost | `Sanctum.CostWood` |
| Stone cost | `Sanctum.CostStone` |
| Upgrade costs | `Sanctum.UpgradeCostWood`, `UpgradeCostStone` |
| Production interval | The `*Interval` constants at the top of `Building.cs` |
| Max level | `Sanctum.MaxLevel` (3) |
| Aura radius | `Sanctum.BaseAuraRadius` |
| Storage cap contribution | `HarvestState.StorageLevels`, `*Cap` properties |

Production intervals are in seconds and are per-building. A shorter interval with
`Capacity = 10` means the building fills up faster and stops, so interval and
capacity must move together.

## Economy

| Lever | Location |
|---|---|
| Starting resources | `HarvestSaveData` field initialisers |
| Resource caps | `HarvestState.*Cap` |
| Feast cost | `HarvestGame.FeastCost` (8, or 4 with a Kitchen) |
| Hymn grace formula | `GameBootstrap.CeremonyRoutine` |
| Sacrifice payoff | `GameBootstrap.CeremonyRoutine` |
| Sharpen cost | `GameBootstrap.SharpenBlade` |
| Meal/medicine/potion recipes | `GameBootstrap` cooking and crafting methods |

## Companions

| Lever | Location |
|---|---|
| Hunger/loyalty/exhaustion rates | `Scripts/Followers/FollowerAgent.cs` |
| Trait effects | `CompanionCatalog.TraitDescription` and `FollowerAgent` |
| Name pool | `CompanionCatalog.Names` (30 names, first 8 are the inner circle) |
| Companion cap | `HarvestGame` recruitment check (`>= 30`) |
| Bed requirement | `Sanctum.IsBed`, `FollowerAgent` sleep logic |

## Creatures and bosses

| Lever | Location |
|---|---|
| Per-archetype speed | `Creature.Init` switch |
| Health and damage | Set at spawn in `Dungeon.cs` |
| Elite modifiers | `Creature.EliteModifier` and the elite application in `Dungeon.cs` |
| Boss phases and attacks | `Scripts/Dungeon/BossController.cs` |
| Boss arena dressing | `Scripts/Dungeon/BossArena.cs` |
| Difficulty multiplier | `Creator.Init` reads `Preferences.CreatureHealthMultiplier` |

## Save compatibility

!!! warning "Changing ids breaks saves"

    Save data stores **ids**, not references. Renaming or removing a weapon,
    rite, heirloom, attire, omen, trait or sanctum id invalidates existing saves
    that reference it. Adding new entries is always safe.

If you must rename an id, add a mapping in `HarvestSaveSystem.Migrate()` that
rewrites the old id to the new one before the rest of the repair pass runs.

## A tuning workflow

1. Change one value.
2. Enter Play mode, not the built player — iteration is faster.
3. Use the smoke test for regression: `.\Tools\build.ps1 -RunSmoke` exercises
   every system and fails on exceptions.
4. Keep a note of the value you changed; the template ships no balance
   spreadsheet.

## Related

- [Economy & resources](../guide/economy.md)
- [Combat & Wardens](../guide/combat.md)
- [Adding content](adding-content.md)
