# Ceremonies & omens

Ceremonies are the ritual layer that ties the settlement economy to progression.
Omens are run-altering modifiers collected from bosses and chests.

## Ceremonies

Open the ceremony menu at the **Harvest Altar** and choose with `1`–`6`. The altar
has a short cooldown between ceremonies (`HarvestAltar.CanPreach`).

| # | Ceremony | Cost | Effect |
|---|---|---|---|
| 1 | **Hymn** | 1 food per companion | Grace scaled by attendance, hunger, traits, sickness and dissent; +3 loyalty and +15 XP per companion |
| 2 | **Feast** | 8 food (4 with a Kitchen) | +20 grace (× grace multiplier), feeds everyone, +10 loyalty, +8 XP |
| 3 | **Sacrifice** | 1 companion (requires more than 3) | +45 grace, +10 pearls |
| 4 | **Sharpen blade** | 15 pearls | +5 permanent weapon damage |
| 5 | **Ceremony of Plenty** | 10 food | +5 meals, +10 grace |
| 6 | **Ceremony of Warding** | 10 pearls | Cures and strengthens every companion |

### How the hymn payout is calculated

The hymn is deliberately the most legible ceremony, and the one worth tuning
first.

```csharp
int baseGain = 5;
foreach (var f in companions)
{
    if (f.Hunger < 30f) { hungry++; baseGain += 1; }
    else baseGain += 2;

    baseGain += f.GraceBonus;                 // Devout trait
    if (f.Sick) baseGain -= 1;
    if (f.Dissenter) baseGain -= 2;
}

int fed = Mathf.Min(HarvestState.Data.food, HarvestState.Data.companions);
HarvestState.Data.food -= fed;
float penalty = fed < HarvestState.Data.companions ? 0.5f : 1f;
int gain = Mathf.RoundToInt(baseGain * HarvestState.GraceMultiplier * penalty);
```

So a well-fed, devout, healthy flock is worth roughly three times a starving,
sick, dissenting one. The `GraceMultiplier` comes from creeds, omens and
heirlooms.

### Creeds

Every third hymn, `OfferCreed` presents a one-time choice of creed. Creeds are
stored as three separate slots — grace, combat, economy — and apply passive
bonuses from `HarvestState` (`GraceMultiplier`, `BonusDamage`, `GatherBonus`,
`BonusPearls`, `BonusFood`).

### Cooldown and interruption

`CeremonyRoutine` locks input, gathers companions at the altar, waits 3.4
seconds, applies the effect, waits 1.6 seconds, then releases everyone. Nothing
else can start while `ceremonyRunning` is true, and saving is blocked.

## Omens

Omens are collected after boss fights and from chests. Each is a permanent
modifier for the current save. The deck has 18 entries:

| Id | Omen | Effect |
|---|---|---|
| `sun` | Dawnfire | +15 maximum health |
| `moon` | Shrouded Tides | +15% grace from ceremonies |
| `star` | Lodestar | +5 melee damage |
| `tower` | Shattered Spire | +1 pearl per slain husk |
| `wheel` | Turning Garden | +1 food per Rose Garden harvest |
| `death` | Last Petal | +10% weapon damage |
| `judgement` | Rite of Rebirth | Rites recharge 25% faster |
| `temperance` | Still Pool | Heal 8 after every cleared room |
| `devil` | Thorned Crown | +12% critical chance and stronger crits |
| `hermit` | Lantern Keeper | +5 potion capacity |
| `hanged` | Suspended Bloom | +15% rite damage |
| `world` | Worldroot | +10% grace from ceremonies |
| `priestess` | Harvest Oracle | +10 maximum health |
| `emperor` | Sovereign's Seal | +2 melee damage |
| `empress` | Harvest Empress | +1 food per Rose Garden harvest |
| `chariot` | Swift Chariot | +5% movement speed |
| `magician` | Ember Magician | +2 bomb capacity |
| `lovers` | Twin Blossoms | +5 maximum health |

The deck lives in one array in `GameBootstrap.cs`:

```csharp
static readonly (string id, string name, string desc)[] OmenDeck =
{
    ("sun", "Dawnfire", "+15 maximum health"),
    // ...
};
```

Adding an omen means adding a tuple and checking `HarvestState.HasOmen(id)`
wherever the effect should apply.

## Heirlooms and attire

Two more permanent layers sit alongside omens:

- **18 heirlooms** (`HeirloomCatalog`) — permanent passives bought in shops or
  found in chests. All owned heirlooms apply at once.
- **11 attire pieces** (`AttireCatalog`) — split into **Armor** and **Charm**
  slots. One of each is equipped at a time; `HarvestState.DamageTakenMultiplier`,
  `RiteCooldownMultiplier` and friends read the equipped ids.

| Armor | Effect | Charm | Effect |
|---|---|---|---|
| Rose Mantle | +12 max health | Cinder Charm | +12% weapon damage |
| Sun Scale | −10% damage taken | Tideglass Charm | Rites recharge 20% faster |
| Thorn Mail | Attackers take 4 damage | Fang Charm | Attacks 8% faster |
| Pearl Plate | +15 max health | Owl Pinion | +5% crit chance |
| Guardian Sigil | −5% damage taken | Serpent Coil | Melee hits heal 1 |
| Silver Mantle | Rites recharge 10% faster | | |

## Related

- [Economy & resources](economy.md)
- [Combat & Wardens](combat.md)
- [Catalogs API](../reference/catalogs.md)
