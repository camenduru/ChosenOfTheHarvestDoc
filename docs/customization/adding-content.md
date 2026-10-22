# Adding content

Every system follows the same shape: a definition class, a `static readonly`
array, and lookup helpers. Adding content means adding an entry to that array.

## Add a weapon

`Scripts/Combat/WeaponCatalog.cs`:

```csharp
public static readonly ArmamentDef FanScythe = new ArmamentDef
{
    Type = ArmamentType.Scythe,
    Id = "fanscythe",
    Name = "Fan Scythe",
    BaseDamage = 32,
    Cooldown = 0.46f,
    Range = 2.4f,
    ArcDot = -0.2f,
    Knockback = 3f,
    ComboCount = 3,
    ComboWindow = 0.5f,
    Lunge = 0.24f,
    BleedDps = 4f,
    BleedDuration = 3f,
    Tint = new Color(0.9f, 0.8f, 1f)
};
```

Then register it:

```csharp
public static readonly ArmamentDef[] All =
{
    Sword, Axe, Dagger, Spear, Hammer, Gauntlets, Scythe, Mace,
    Flail, Rapier, Cleaver, Halberd, FanScythe
};
```

`Q` cycles through `All`, and `HarvestState.Data.weaponType` stores the `Id`.
The weapon is unlocked immediately because cycling has no ownership gate; add
one in `CycleWeapon` if you want unlocks.

## Add a rite

`Scripts/Combat/SpellCatalog.cs`:

```csharp
public static readonly RiteDefinition Bloom = new RiteDefinition
{
    Id = "bloom",
    Name = "Bloom",
    Description = "a burst of healing pollen",
    Cooldown = 10f,
    Damage = 10,
    Radius = 3.8f,
    Knockback = 1f,
    HealPerHit = 8,
    Color = new Color(0.7f, 1f, 0.7f)
};
```

Add it to `RiteCatalog.All`. `Z` cycles rites and `X` casts.

## Add an omen

`Scripts/Core/GameBootstrap.cs`:

```csharp
("lantern", "Hearth Lantern", "+10% grace and +5 food per harvest"),
```

Then implement the effect in `GameState.cs`, e.g. inside `GraceMultiplier`:

```csharp
public static float GraceMultiplier =>
    (Data.creedGrace == "feast" ? 1.5f : 1f) *
    (HasOmen("moon") ? 1.15f : 1f) *
    (HasOmen("lantern") ? 1.1f : 1f) * ...
```

!!! warning

    Omen ids are written into saves. Adding is safe; renaming is not.

## Add a sanctum type

Four edits, in order:

1. **Enum** — `Scripts/Buildings/Building.cs`:

```csharp
public enum SanctumType
{
    Farm, Shrine, Bed, Lumberyard, Quarry, Kitchen,
    Storage, HerbGarden, Apothecary, TeaHouse, Memorial, Reflection,
    SunBath, Obelisk, Temple, Sawmill, Stoneworks, WheatField,
    BellTower, Reliquary, Orchard
}
```

2. **Costs** — add cases to `CostWood`, `CostStone`, and optionally
`UpgradeCostWood`/`UpgradeCostStone`.

3. **Label and localisation** — add a case to `Sanctum.Label`'s switch and a
`building.orchard` key to `en.json`. (The label falls back to the enum name, so
this is cosmetic.)

4. **Sprite** — add `Resources/Art/Sanctums/orchard.png`. The loader is:

```csharp
string art = type.ToString().ToLowerInvariant();
var sprite = Resources.Load<Sprite>("Art/Sanctums/" + art);
```

!!! danger "The enum member and the file name must match"

    `SpriteForSanctum` derives the file name from the enum member. If you add
    `Orchard` and forget `orchard.png`, the building silently falls back to the
    shrine sprite.

Also add the build-menu entry where the game lists buildable types
(`GameBootstrap.ShowBuildMenu` / `BuildSummary`) and a height case in
`TargetHeightForSanctum`.

## Add a follower trait

`Scripts/Followers/FollowerCatalog.cs`:

```csharp
public enum CompanionTrait { Loyal, Hardworking, Glutton, Devout, Sickly, Lazy, Curious }
```

Add it to `Traits`, then give it a name and description case. Finally implement
the behaviour in `FollowerAgent` where the other traits are read.

## Add a creature archetype

`Scripts/Dungeon/Enemy.cs`:

```csharp
public enum Archetype
{
    Melee, Fast, Brute, Caster, Charger, Exploder,
    Shielder, Summoner, Healer, Archer, Slammer, Assassin, Howler
}
```

Then:

1. add a speed case in `Creature.Init`;
2. add a behaviour branch in `Creature.Update` (the existing archetypes show the
   pattern: `charger`, `summoner`, `healer`, `slammer`, `assassin`);
3. add the archetype to the spawn table in `Dungeon.cs`.

## Add a quest stage

`Scripts/Core/QuestSystem.cs`:

```csharp
static readonly string[] Keys =
{
    "quest.buildfarm", "quest.hymn", "quest.reflection",
    "quest.forest", "quest.temple", "quest.sanctum",
    "quest.companions", "quest.devotion", "quest.orchard", "quest.done"
};
```

Update `FinalStage`, add a `case` in the `Evaluate` switch with the completion
condition, and add `quest.orchard` to `en.json`. Rewards are automatic: stages
0–4 pay 2 devotion, stages 5+ pay 4.

## Add a localisation key

`Resources/Localization/en.json`:

```json
{
  "key": "hud.orchard",
  "value": "Orchard {0}/{1}"
}
```

Read it with `Lexicon.Get("hud.orchard", "Orchard")` or
`Lexicon.Format("hud.orchard", "Orchard {0}/{1}", a, b)`. Add the same key to
`xx.json` if you want the pseudo-locale to cover it (any placeholder value is
fine — it exists to test layout).

## Add a save field

1. Add the field to `HarvestSaveData` with a default:

```csharp
public int orchardYield = 1;
```

2. Bump the schema:

```csharp
// Scripts/Core/SaveSystem.cs
public const int CurrentVersion = 5;
```

3. Extend the repair pass in `Migrate()` so old saves get a sane value:

```csharp
if (HarvestState.Data.orchardYield < 1)
    HarvestState.Data.orchardYield = 1;
```

`Migrate()` is an idempotent repair pass rather than a per-version chain: it runs
after every load and fixes any field that is out of range. That keeps old saves
loadable without writing one migration method per version.

## Related

- [Catalogs API](../reference/catalogs.md)
- [Balancing & tuning](balancing.md)
- [State & save schema](../architecture/state-and-save.md)
