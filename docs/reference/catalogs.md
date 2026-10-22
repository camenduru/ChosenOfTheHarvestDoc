# Catalogs API reference

Every number the player can feel — weapon rhythm, rite damage, heirloom bonuses,
armour effects, companion traits, building costs — lives in a static catalog
class rather than in the systems that use them.

## How the catalogs work

- A catalog is a `public static class` with `readonly` entries and a `Get(id)`
  lookup; entries are plain C# objects, not `ScriptableObject`s.
- The equipped identity is a string id on `HarvestSaveData` (`weaponType`,
  `riteType`, `armorId`, `charmId`), resolved through the catalog on demand.
- Unknown ids never throw: weapons and rites fall back to the first entry,
  heirlooms and attire to `null`.
- Because `HarvestState` computes bonuses from the same ids, adding an entry
  requires adding its effect in `GameState.cs`; the table alone is not enough.

## Types

### ArmamentCatalog

`Combat/WeaponCatalog.cs` — twelve armaments, each with its own rhythm, arc and
on-hit effect.

**Entries**

| Id | Name | Damage | Cooldown | Range | Combo | On-hit effect |
|---|---|---|---|---|---|---|
| `sword` | Rose Saber | 25 | 0.34 | 1.9 | 3 | none; balanced arc (`0.15`) |
| `axe` | Golden Axe | 42 | 0.58 | 2.05 | 1 | slow ×0.55 for 1.3 s |
| `dagger` | Fan Blade | 13 | 0.19 | 1.45 | 4 | bleed 3 dps for 3.5 s |
| `spear` | Silk Lance | 30 | 0.42 | 2.65 | 2 | none; narrow arc (`0.55`) |
| `hammer` | Bell Hammer | 52 | 0.88 | 2.15 | 1 | slow ×0.45 for 1.6 s, hits all |
| `gauntlets` | Thorn Claws | 9 | 0.14 | 1.25 | 5 | lifesteal 1 |
| `scythe` | Crescent Scythe | 36 | 0.50 | 2.5 | 2 | bleed 2 dps for 3 s |
| `mace` | Sunburst Mace | 58 | 1.00 | 2.0 | 1 | slow ×0.4 for 2 s, hits all |
| `flail` | Chain of Petals | 38 | 0.72 | 2.35 | 2 | slow ×0.5 for 1.4 s, hits all |
| `rapier` | Glass Rapier | 20 | 0.22 | 2.2 | 4 | none; narrow thrust (`0.5` arc) |
| `cleaver` | Jade Cleaver | 46 | 0.66 | 1.85 | 2 | bleed 5 dps for 4 s |
| `halberd` | Royal Halberd | 40 | 0.55 | 2.8 | 2 | none; longest reach (`2.8`) |

**Members**

| Member | Type | Description |
|---|---|---|
| `All` | `ArmamentDef[]` | All twelve, in enum order |
| `Sword` … `Halberd` | `ArmamentDef` | Named static entries |
| `Current` | `ArmamentDef` | `Get(HarvestState.Data.weaponType)` |
| `Get(string id)` | `ArmamentDef` | Exact id match, otherwise `Sword` |
| `Next()` | `ArmamentDef` | Wraps to the next entry in `All` |

```csharp
HarvestState.Data.weaponType = "hammer";      // choosing by id
int damage = HarvestState.WeaponDamage;        // resolves the catalog for you
```

### ArmamentDef

| Field | Type | Description |
|---|---|---|
| `Type` | `ArmamentType` | Enum tag (`Sword` … `Halberd`) |
| `Id` | `string` | Save id |
| `Name` | `string` | Display name |
| `BaseDamage` | `int` | Base before `HarvestState.WeaponDamage` bonuses |
| `Cooldown` | `float` | Seconds between swings, scaled by `AttackCooldownMultiplier` |
| `Range` | `float` | Hit radius; the check allows `Range + 0.35` |
| `ArcDot` | `float` | Minimum facing dot; higher is narrower, `-1` is a full circle |
| `Knockback` | `float` | Impulse applied to each target |
| `ComboCount` | `int` | Swing cycle length |
| `ComboWindow` | `float` | Seconds before the combo resets |
| `Lunge` | `float` | Forward step per swing |
| `BleedDps` / `BleedDuration` | `float` | Damage-over-time applied on hit |
| `SlowFactor` / `SlowDuration` | `float` | Slow multiplier and its duration |
| `HitsAll` | `bool` | Skips the arc test and hits every target in range |
| `Lifesteal` | `int` | Health restored per target hit |
| `Tint` | `Color` | Swing VFX colour |

Combo swings add `+15%` damage per combo index (`comboIndex * 0.15`); the
critical roll happens once for the whole swing, not per target.

### RiteCatalog

`Combat/SpellCatalog.cs` — three rites, cast with X and cycled with Z. All are
available from the start.

**Entries**

| Id | Name | Cooldown | Damage | Radius | Knockback | Extra |
|---|---|---|---|---|---|---|
| `smite` | Sunburst | 7.5 | 34 | 3.5 | 4.5 | none |
| `frost` | Shroud of Frost | 8.5 | 18 | 4.1 | 1.5 | slow ×0.42 for 2.8 s |
| `blood` | Crimson Vow | 9.5 | 22 | 3.2 | 2.0 | heals 5 per target hit |

**Members**

| Member | Type | Description |
|---|---|---|
| `All` | `RiteDefinition[]` | `Smite`, `Frost`, `Blood` |
| `Smite` / `Frost` / `Blood` | `RiteDefinition` | Named entries |
| `Current` | `RiteDefinition` | `Get(HarvestState.Data.riteType)` |
| `Get(string id)` | `RiteDefinition` | Falls back to `Smite` |
| `Next()` | `RiteDefinition` | Wraps the array |

### RiteDefinition

| Field | Type | Description |
|---|---|---|
| `Id` / `Name` / `Description` | `string` | Identity and menu copy |
| `Cooldown` | `float` | Seconds, scaled by `HarvestState.RiteCooldownMultiplier` |
| `Damage` | `int` | Base damage, scaled by `RiteDamageMultiplier` |
| `Radius` | `float` | Effect radius around the player |
| `Knockback` | `float` | Impulse applied to targets |
| `SlowFactor` / `SlowDuration` | `float` | Optional slow |
| `HealPerHit` | `int` | Health restored per target hit |
| `Color` | `Color` | VFX colour |

### HeirloomCatalog

`Combat/RelicCatalog.cs` — eighteen permanent passives. Every owned heirloom
applies at once, so the whole list in `Data.heirlooms` matters.

**Entries**

| Id | Name | Effect |
|---|---|---|
| `heart` | Ember Heart | +20 maximum health |
| `fang` | Thorn Fang | +5 melee damage |
| `feather` | Zephyr Plume | +12% movement speed |
| `ward` | Warding Opal | take 15% less damage |
| `chalice` | Cup of Stars | +10% grace from ceremonies |
| `ledger` | Gilded Ledger | +1 pearl per kill |
| `harvest` | Orchard Charm | +1 food per farm harvest |
| `mirror` | Obsidian Mirror | +25% rite damage |
| `bell` | Mourning Bell | rites recharge 10% faster |
| `crown` | Sovereign's Diadem | +15% devotion from pilgrimages |
| `mask` | Shrouded Mask | +5% critical chance |
| `hourglass` | Amber Hourglass | attacks 10% faster |
| `lantern` | Pale Lantern | +2 bomb capacity |
| `candle` | Vigil Candle | +10 maximum health |
| `idol` | Cracked Idol | +10% grace |
| `cloak` | Shroud Cloak | take 5% less damage |
| `tome` | Starledger | +10% rite damage |
| `ring` | Sunring | +5% movement speed |

**Members**

| Member | Type | Description |
|---|---|---|
| `All` | `HeirloomDefinition[]` | All eighteen |
| `Get(string id)` | `HeirloomDefinition` | `null` when unknown |
| `NameOf` / `DescriptionOf(string id)` | `string` | Display helpers with fallbacks |
| `ColorOf(string id)` | `Color` | `Color.white` when unknown |
| `RandomUnowned()` | `HeirloomDefinition` | Random not in `Data.heirlooms`, or `null` |

!!! note "`crown` is read outside the catalog"
    `WardenDefeated` multiplies the act reward by `1.15` when
    `HarvestState.HasHeirloom("crown")`; the rest are read inside
    `HarvestState`'s derived properties.

### HeirloomDefinition

Plain data: every bonus it grants is read from its id inside `HarvestState`.

| Member | Type | Description |
|---|---|---|
| `Id` / `Name` / `Description` | `string` | Save id, display name, menu copy |
| `Color` | `Color` | Menu accent colour |

### AttireCatalog

`Combat/GearCatalog.cs` — eleven pieces across two slots. All owned pieces stay
in `Data.attire`; one per slot is equipped.

**Entries**

| Id | Name | Slot | Effect |
|---|---|---|---|
| `wool` | Rose Mantle | Armor | +12 maximum health |
| `scale` | Sun Scale | Armor | take 10% less damage |
| `briar` | Thorn Mail | Armor | attackers take 4 damage |
| `pearl` | Pearl Plate | Armor | +15 maximum health |
| `guard` | Guardian Sigil | Armor | take 5% less damage |
| `moon` | Silver Mantle | Armor | rites recharge 10% faster |
| `ember` | Cinder Charm | Charm | +12% weapon damage |
| `tide` | Tideglass Charm | Charm | rites recharge 20% faster |
| `wolf` | Fang Charm | Charm | attacks 8% faster |
| `owl` | Owl Pinion | Charm | +5% critical chance |
| `serpent` | Serpent Coil | Charm | melee hits heal 1 health |

**Members**

| Member | Type | Description |
|---|---|---|
| `All` | `AttireDefinition[]` | All eleven |
| `Get(string id)` | `AttireDefinition` | `null` when unknown |
| `RandomUnowned()` | `AttireDefinition` | Random not owned, or `null` |
| `Owned(AttireSlot slot)` | `List<AttireDefinition>` | Owned entries of one slot |
| `NameOf` / `DescriptionOf(string id)` | `string` | Display helpers |

Equipping goes through state, not the catalog: `HarvestState.AddAttire("moon")`
returns `false` if the id is unknown or already owned, and `EquipAttire` routes
the piece to `armorId` or `charmId` by its slot.

### AttireSlot

`enum AttireSlot { Armor, Charm }` — two slots; `EnsureLoadout` keeps one valid
owned item in each.

| Value | Save field | Description |
|---|---|---|
| `Armor` | `armorId` | Body slot, defaults to `wool` |
| `Charm` | `charmId` | Trinket slot, defaults to `ember` |

### AttireDefinition

One owned piece: the id drives the effect inside `HarvestState`, the slot decides
which save field it occupies.

| Member | Type | Description |
|---|---|---|
| `Id` / `Name` / `Description` | `string` | Save id, display name, menu copy |
| `Slot` | `AttireSlot` | Which save field it equips into |
| `Color` | `Color` | Menu accent colour |

### CompanionCatalog

`Followers/FollowerCatalog.cs` — names, traits and tints. The first eight names
are the "inner circle", handed out before the wider pool because they own
portrait art.

| Member | Type | Description |
|---|---|---|
| `InnerCircleCount` | `const int` | `8` |
| `InnerCircleName(int index)` | `string` | Name at `index`, or `null` out of range |
| `Traits` | `CompanionTrait[]` | The six traits |
| `RandomName()` | `string` | Uniform pick from the 30-name pool |
| `RandomTrait()` | `CompanionTrait` | Uniform pick from `Traits` |
| `TraitName(CompanionTrait)` | `string` | Localised `trait.<name>` |
| `TraitDescription(CompanionTrait)` | `string` | Localised `trait.<name>.desc` |
| `Parse(string id)` | `CompanionTrait` | Name match, otherwise `Loyal` |
| `RandomTint()` | `Color` | Lerp between rose/gold and teal |

The 30-name pool, in order: Amara, Sable, Iris, Nadia, Yara, Liora, Mira, Yafit,
Calista, Delphine, Esme, Faye, Gemma, Helia, Isolde, Jade, Kira, Lyra, Maeve,
Nerissa, Opal, Perla, Quinn, Rhea, Seraphine, Talia, Uma, Vesper, Wren, Yvonne.
Only those eight are treated as named (white tint) in the hub; `Amara` has no
portrait file and falls back to the shared `companion` sprite, everyone else is
tinted.

### CompanionTrait

`enum CompanionTrait { Loyal, Hardworking, Glutton, Devout, Sickly, Lazy }`.

| Trait | Description fallback |
|---|---|
| `Loyal` | loyalty fades slowly |
| `Hardworking` | steadier worker, +2 output |
| `Glutton` | hungers quickly |
| `Devout` | +2 grace at hymns |
| `Sickly` | falls ill easily |
| `Lazy` | works slowly, tires quickly |

### SanctumType

`Buildings/Building.cs` — twenty structures. Costs are static `Sanctum` methods;
production intervals are private constants on the same class.

| `SanctumType` | Wood | Stone | Effect |
|---|---|---|---|
| `Farm` | 20 | 0 | +2 food every 12 s |
| `Shrine` | 0 | 25 | +1 grace every 8 s, scaled by dirt |
| `Bed` | 15 | 0 | +1 companion and a bed slot |
| `Lumberyard` | 30 | 0 | +1 wood every 10 s |
| `Quarry` | 20 | 20 | +1 stone every 12 s |
| `Kitchen` | 25 | 10 | worker turns 2 food into 1 meal every 16 s |
| `Storage` | 30 | 10 | +100 wood/stone and +50 food caps per level |
| `HerbGarden` | 20 | 0 | +1 herb every 14 s |
| `Apothecary` | 30 | 20 | worker turns 2 herbs into 1 medicine every 18 s |
| `TeaHouse` | 30 | 10 | social aura radius 5.2 |
| `Memorial` | 25 | 10 | memorials and mourning, aura 3.2 |
| `Reflection` | 40 | 20 | holds dissenters until loyalty returns |
| `SunBath` | 25 | 15 | rest and loyalty recovery, aura 4.5 |
| `Obelisk` | 15 | 30 | +1 grace every 12 s, aura 5.5 |
| `Temple` | 50 | 30 | +2 grace every 12 s, aura 6 |
| `Sawmill` | 35 | 10 | +2 wood every 9 s |
| `Stoneworks` | 30 | 20 | +2 stone every 10 s |
| `WheatField` | 25 | 5 | +2 food every 10 s |
| `BellTower` | 40 | 35 | +1 grace every 15 s, aura 7 |
| `Reliquary` | 30 | 25 | +1 pearl every 18 s, aura 3.5 |

| Member | Type | Description |
|---|---|---|
| `CostWood(SanctumType)` | `int` | Build cost, `0` for `Shrine` |
| `CostStone(SanctumType)` | `int` | Build cost |
| `UpgradeCostWood(SanctumType)` | `int` | `round(0.6 × wood) + 10` |
| `UpgradeCostStone(SanctumType)` | `int` | `round(0.6 × stone) + 8` |

```csharp
int wood = Sanctum.CostWood(SanctumType.Temple);        // 50
int next = Sanctum.UpgradeCostWood(SanctumType.Temple); // 40
```

Production is `(base + workerBonus + levelBonus) × conditionScale`, where the
worker bonus is `CompanionAgent.WorkBonus` (2 for `Hardworking`, else 1, plus
`(level - 1) / 2`, plus 1 during a meal buff) and `conditionScale` is
`Lerp(0.55, 1, Condition / 100)`. Storage levels feed `HarvestState.StorageLevels`,
raising every cap.

### OmenDeck

The 18-card omen deck is **private**: `static readonly (string id, string name,
string desc)[] OmenDeck` inside `HarvestGame`. It is not accessible from other
classes; grant cards with `HarvestGame.GrantOmen(id, name)`, offer them with
`ShowOmenMenu`, and browse them through `ShowCatalogSection(1)`.

| Id | Name | Effect |
|---|---|---|
| `sun` | Dawnfire | +15 maximum health |
| `moon` | Shrouded Tides | +15% grace from ceremonies |
| `star` | Lodestar | +5 melee damage |
| `tower` | Shattered Spire | +1 pearl per slain husk |
| `wheel` | Turning Garden | +1 food per rose garden harvest |
| `death` | Last Petal | +10% weapon damage |
| `judgement` | Rite of Rebirth | rites recharge 25% faster |
| `temperance` | Still Pool | heal 8 after every cleared room |
| `devil` | Thorned Crown | +12% critical chance and stronger crits |
| `hermit` | Lantern Keeper | +5 potion capacity |
| `hanged` | Suspended Bloom | +15% rite damage |
| `world` | Worldroot | +10% grace from ceremonies |
| `priestess` | Harvest Oracle | +10 maximum health |
| `emperor` | Sovereign's Seal | +2 melee damage |
| `empress` | Harvest Empress | +1 food per rose garden harvest |
| `chariot` | Swift Chariot | +5% movement speed |
| `magician` | Ember Magician | +2 bomb capacity |
| `lovers` | Twin Blossoms | +5 maximum health |

`HarvestState.OmenSummary()` maps every id to its display name, and `HasOmen(id)`
is the membership test used by the bonus formulas. If every card is owned,
`ShowOmenMenu` pays `+25 pearls` instead of offering a choice.

!!! warning "Adding a card takes three edits"
    Add the tuple to `OmenDeck`, add the id to `OmenSummary`'s switch (otherwise
    the HUD prints the raw id), and add the effect to the relevant derived
    property in `GameState.cs`. A card with no formula is a dead card.

## Related

- [Core API reference](core.md)
- [Systems map](../architecture/systems-map.md)
- [Balancing & tuning](../customization/balancing.md)
- [Adding content](../customization/adding-content.md)
