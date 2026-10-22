# Economy & resources

The Harvesthold runs on twelve tracked resources. Every producer, cost and cap
lives in code, so tuning the economy is a matter of editing a constant.

## Resources

| Resource | Produced by | Spent on |
|---|---|---|
| **Wood** | Lumberyard, Timber Mill | Building, upgrading |
| **Stone** | Quarry, Stoneworks | Building, upgrading |
| **Grace** | Sunwell, ceremonies | Ceremonies, sacrifices, recipes |
| **Pearls** | Combat kills, chests, sacrifice | Sharpening, warding, shop stock |
| **Food** | Rose Garden, Wheat Field, gathering | Hymns, feasts, feeding companions |
| **Meals** | Kitchen, Ceremony of Plenty | Feeding companions efficiently |
| **Herbs** | Herb Terrace | Medicine |
| **Medicine** | Apothecary | Curing sick companions |
| **Potions** | Apothecary | Healing the player |
| **Bombs** | Crafting | Combat |
| **Hymns** | Ceremonies | Quest progression, creed offers |
| **Devotion** | Quests, pilgrimages, bosses | Permanent meta-upgrades |

## Storage caps

Caps are enforced centrally in `HarvestState.NotifyChanged`, so every producer
obeys them automatically.

| Resource | Cap |
|---|---|
| Wood | `250 + storageLevels × 100` |
| Stone | `250 + storageLevels × 100` |
| Food | `100 + storageLevels × 50` |
| Herbs | `30 + storageLevels × 20` |
| Meals | `30 + storageLevels × 20` |
| Medicine | `10 + storageLevels × 5` |
| Bombs | `5 + storageLevels × 2 + omen/heirloom bonuses` |
| Potions | `20 + 5 if the Lantern Keeper omen is held` |

`storageLevels` is the sum of the levels of every **Vault** (Storage) sanctum.
That makes storage the one building that scales indefinitely in value.

```csharp
// GameState.cs
public static int StorageLevels
{
    get
    {
        int levels = 0;
        foreach (var building in Data.buildings)
            if (string.Equals(building.type, "Storage", StringComparison.Ordinal))
                levels += Mathf.Max(1, building.level);
        return levels;
    }
}
```

## Production

Producing sanctums accumulate output on a timer and store up to 10 units before
they stop. A worker assigned to the sanctum is required for the listed interval;
without one the rate is slower.

| Sanctum | Output | Interval |
|---|---|---|
| Rose Garden (Farm) | Food | 12 s |
| Sunwell (Shrine) | Grace | 8 s |
| Timber Grove (Lumberyard) | Wood | 10 s |
| Marble Quarry (Quarry) | Stone | 12 s |
| Kitchen Pavilion | Meals | 16 s |
| Herb Terrace | Herbs | 14 s |
| Elixir Atelier (Apothecary) | Medicine, potions | 18 s |
| Timber Mill (Sawmill) | Wood | 9 s |
| Stoneworks | Stone | 10 s |
| Wheat Field | Food | 10 s |
| Bell Tower | Grace | 15 s |
| Reliquary | Pearls | 18 s |

!!! tip "Auto-haul"

    Companions with the hauling job collect from producing sanctums and deposit
    into storage automatically. Manual collection (`E` on the building) is always
    faster but costs your attention.

## Food and meals

Feeding is the core pressure on the economy.

- A **hymn** feeds one food per companion (up to the number of companions).
  If there is not enough food, the grace gain is halved.
- A **feast** feeds everyone and grants +10 loyalty each. It costs **8 food**, or
  **4** if you have built a Kitchen Pavilion.
- **Meals** are the efficient food unit: a meal feeds a companion without
  consuming raw food. `MealCap` limits how many you can stockpile.

## Pearls

Pearls are the combat currency. They come from kills, chests, boss rewards and
the Sacrifice ceremony. They are spent on:

- **Sharpen blade** — 15 pearls for +5 permanent weapon damage (repeatable);
- **Ceremony of Warding** — 10 pearls to cure and strengthen everyone;
- shop stock during expeditions.

Because sharpening is unlimited, pearls are the main long-run power sink. If you
rebalance the economy, change the pearl drop rate and the sharpen cost together.

## Devotion

Devotion is the meta-currency. It survives death and is spent on permanent
upgrades from the pilgrimage map.

| Upgrade | Effect |
|---|---|
| **Vitality** | +15 maximum health |
| **Wrath** | +4 damage |
| **Provisions** | +10 food and +5 meals immediately |
| **Charisma** | New companions start with +3 loyalty |

Devotion is earned from:

- quest completion (2 devotion per early stage, 4 per late stage);
- boss kills (8 / 12 / 16 by act);
- pilgrimages;
- the `Sovereign's Diadem` heirloom (+15% devotion from pilgrimages).

## Shop rooms

Expeditions contain shop rooms where pearls buy heirlooms and attire. Stock is
drawn from `HeirloomCatalog.RandomUnowned()` and
`AttireCatalog.RandomUnowned()`, so once everything is owned, shops stop
offering gear and become pearl sinks for consumables.

## Related

- [Building & sanctums](building.md)
- [Ceremonies & omens](ceremonies.md)
- [Balancing & tuning](../customization/balancing.md)
