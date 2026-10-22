# Localisation key reference

Every player-facing string in the template is addressed by a dotted key and resolved
through the `Lexicon` static class from a JSON table in
`Assets/ChosenOfTheHarvest/Resources/Localization`. This page documents the key
convention, lists all 349 English keys grouped by prefix, and explains the
`?XX?`-prefixed pseudo-locale used for layout testing.

## How Lexicon resolves a string

| Member | Returns | Description |
|---|---|---|
| `Locale` | `string` | Active locale id, `"en"` until `SetLocale` |
| `Get(string key, string fallback = null)` | `string` | Table value, else `fallback`, else the key itself |
| `Format(string key, string fallback, params object[] args)` | `string` | `Get` then `string.Format`; a broken template returns unformatted text |
| `KeyCount` | `int` | Entries loaded for the active locale |
| `SetLocale(string next)` | `void` | Switches locale, clears the cache, reloads on next lookup |

The table is `Resources/Localization/<locale>.json`, loaded once and cached until
the locale changes. A missing file leaves the table empty rather than throwing, so
every string falls back to its in-code default. The file shape is fixed by
`JsonUtility`:

```json
{
  "entries": [
    { "key": "title.name", "value": "CHOSEN OF THE HARVEST" }
  ]
}
```

`Lexicon.Get` is also used with a literal default at the call site, which is why
code reads like `Lexicon.Get("dungeon.locked", "The door is locked …")` — that
second argument is the fallback, not a translation.

!!! warning "Duplicate keys: last one wins"
    `menu.assign.title` is defined twice in `en.json` — once as `ASSIGN` and later as
    `ASSIGN {0}`. The loader assigns `Values[key] = value` in file order, so the later
    entry replaces the earlier one. Remove the first entry when you translate.

!!! note "Where the locale is chosen"
    `Preferences.Language` is stored in PlayerPrefs as `choth_settings_language`.
    Settings page 2 cycles English ⇄ Pseudo, and `HarvestGame` applies it through
    `Lexicon.SetLocale` when the game starts.

## Key naming convention

| Rule | Example |
|---|---|
| `<system>.<subject>[.<detail>]`, lowercase, dot separated | `quest.hymn`, `trait.lazy.desc` |
| Compound subjects are concatenated, never hyphenated | `dungeon.treasuretaken`, `menu.actions.reflection` |
| Screen and menu titles are ALL CAPS | `campaign.title`, `shop.sell.title` |
| Toast keys mirror the action that produced them | `companion.blessed`, `slots.deleted` |
| Input prompts are prefixed with the key glyph | `[E]  Descend deeper`, `[B]  Build` |
| Fields are separated by a spaced bullet `  •  ` | `ACT I  •  ASHEN ORCHARD  •  The Gilded Stag` |
| Placeholders are positional `{0}`–`{3}`, filled by `Lexicon.Format` | `shop.heirloom` = `{0}   •   25 pearls   •   stock {1}` |
| Ellipses are three ASCII dots, never `…` | `The rose garden is growing...` |

Four prefixes are composed at runtime from an enum name, so the key is never a
literal in code:

| Composed key | Source | Values |
|---|---|---|
| `building.<type>` | `Sanctum.Label`, `type.ToString().ToLowerInvariant()` | 20 building names |
| `room.<type>` | `FloorPlanGenerator.Name(type)` | `entrance` … `forge` |
| `trait.<trait>` / `trait.<trait>.desc` | `CompanionCatalog` | 6 traits × 2 |
| `phase.<phase>` | `SeasonCycle.PhaseName` | `dawn`, `day`, `dusk`, `night` |

`campaign.act1`–`campaign.act3` sit in a fixed array, and `quest.*` / `npc.hint.*`
are referenced from ordered lists in `QuestSystem` and `HarvestGame`, so their key
names are positional: reordering those lists reorders the story.

## Coverage

349 keys in 30 prefixes. The groups below follow the order a player meets them.

| Prefix | Keys | Purpose |
|---|---|---|
| `title.*` | 11 | Title screen and prologue |
| `campaign.*` | 21 | Pilgrimage map, act cards, victory and defeat |
| `meta.*` | 8 | Permanent Devotion upgrades |
| `quest.*` | 11 | Quest log entries and rewards |
| `npc.*` | 16 | The Oracle's intro, hints and progress line |
| `act.*` | 3 | Act intro, clear and reward lines |
| `hud.*` | 7 | Persistent HUD rows |
| `phase.*` | 4 | Time-of-day names |
| `prompt.*` | 19 | World interaction prompts |
| `building.*` | 20 | Sanctum display names |
| `room.*` | 10 | Dungeon room labels |
| `menu.*` | 36 | Build, companion, duty, bed, store and assign menus |
| `companion.*` | 17 | Companion actions, relationships and results |
| `trait.*` | 12 | Trait names and descriptions |
| `mood.*` | 9 | Mood labels |
| `priority.*` | 3 | Duty priorities |
| `ceremony.*` | 10 | Altar ceremonies and costs |
| `daily.*` | 7 | Daily event toasts |
| `creed.*` | 3 | Creed adoption |
| `dungeon.*` | 31 | Objectives, prompts and corridor lines |
| `shop.*` | 22 | Merchant stock, prices and selling |
| `forge.*` | 3 | Forge menu |
| `curse.*` | 3 | Omen altar |
| `recipe.*` | 10 | Kitchen and apothecary recipes |
| `toast.*` | 11 | Short confirmations |
| `pause.*` | 7 | Pause menu |
| `settings.*` | 15 | Preferences |
| `slots.*` | 14 | Save slots |
| `rebind.*` | 5 | Key rebinding |
| `photo.*` | 1 | Photo mode hint |

### `title.*`

| Key | English value |
|---|---|
| `title.name` | CHOSEN OF THE HARVEST |
| `title.subtitle` | A CLEAN-ROOM STUDY BUILD |
| `title.new` | New game |
| `title.continue` | Continue |
| `title.load` | Load game |
| `title.settings` | Preferences |
| `title.quit` | Quit |
| `title.empty` | No save yet |
| `title.prologue.1` | The old grace is broken. A Sovereign wears a crown. |
| `title.prologue.2` | Gather a harvest. Grow the Harvesthold. Break the three wardens. |
| `title.prologue.3` | The Oracle waits by the altar. |

### `campaign.*`

| Key | English value |
|---|---|
| `campaign.title` | PILGRIMAGE MAP |
| `campaign.subtitle` | Three wardens bar the road. Free their captives and break them in order. |
| `campaign.act1` | ACT I  •  ASHEN ORCHARD  •  The Gilded Stag |
| `campaign.act2` | ACT II  •  DROWNED CONSERVATORY  •  The Mire Queen |
| `campaign.act3` | ACT III  •  EMBER CITADEL  •  The Obsidian Sultan |
| `campaign.cleared` | CLEARED |
| `campaign.available` | AVAILABLE |
| `campaign.locked` | LOCKED |
| `campaign.meta` | Devotion upgrades  •  {0} devotion |
| `campaign.quests` | Quest log |
| `campaign.close` | Close |
| `campaign.begin` | Begin pilgrimage |
| `campaign.return` | Return to the Harvesthold |
| `campaign.locked.toast` | Clear the previous warden first |
| `campaign.victory.title` | THE HARVEST ENDURES |
| `campaign.victory.body` | All three wardens have fallen. The Harvesthold blooms, and its rescued companions are free. |
| `campaign.victory.reward` | The Harvesthold is eternal. Devotion flows freely. |
| `campaign.victory.toast` | You return to the Harvesthold, victorious |
| `campaign.defeat.toast` | You were carried back to the harvest |
| `campaign.death.pearls` | You fell in the pilgrimage   •   lost {0} pearls |
| `campaign.death.wake` | You collapse, then wake at the foot of the altar |

### `meta.*`

| Key | English value |
|---|---|
| `meta.title` | PERMANENT DEVOTION |
| `meta.vitality` | Vitality  •  +15 max health  •  {0} devotion |
| `meta.wrath` | Wrath  •  +4 damage  •  {0} devotion |
| `meta.provisions` | Provisions  •  +10 food and +5 meals now  •  {0} devotion |
| `meta.charisma` | Charisma  •  new companions start with +3 loyalty  •  {0} devotion |
| `meta.back` | Back |
| `meta.max` | already mastered |
| `meta.poor` | Not enough devotion |

### `quest.*`

| Key | English value |
|---|---|
| `quest.title` | QUEST LOG |
| `quest.buildfarm` | Plant a Rose Garden |
| `quest.hymn` | Hold a hymn at the Harvest Altar |
| `quest.reflection` | Defeat the Gilded Stag in the Ashen Orchard |
| `quest.forest` | Defeat the Mire Queen in the Drowned Conservatory |
| `quest.temple` | Defeat the Obsidian Sultan in the Ember Citadel |
| `quest.done` | The Harvesthold has endured all three wardens. |
| `quest.reward` | Quest complete  •  +{0} devotion |
| `quest.sanctum` | Raise the Grand Temple |
| `quest.companions` | Grow the flock to eight |
| `quest.devotion` | Reach 30 Devotion |

### `npc.*`

| Key | English value |
|---|---|
| `npc.name` | THE ORACLE |
| `npc.intro.1` | The Harvestsworn Sovereign returns with an empty crown and a garden waiting to bloom. |
| `npc.intro.2` | Plant a Rose Garden. Hold a hymn at the Harvest Altar. Then open the portal and free the first captive. |
| `npc.intro.3` | Grace is permanent. Spend Devotion on the pilgrimage map. Every rescued companion remembers. |
| `npc.hint.farm` | The Rose Garden is the first heartbeat of the Harvesthold. Press B, choose it, and place it on the grid. |
| `npc.hint.hymn` | Your companions need words. Stand at the Harvest Altar and hold a hymn. |
| `npc.hint.reflection` | The Ashen Orchard waits beyond the portal. The Gilded Stag guards the first road. |
| `npc.hint.forest` | The Drowned Conservatory has swallowed the second road. Its Mire Queen is older and angrier. |
| `npc.hint.temple` | The Ember Citadel is the last road. End the Obsidian Sultan, and the harvest will have a future. |
| `npc.hint.done` | Three roads, three wardens, one harvest. You have done what the old world could not. |
| `npc.hint.sanctum` | A Grand Temple would anchor the Harvesthold. Build one. |
| `npc.hint.companions` | Eight souls make a flock. Recruit or rescue more. |
| `npc.hint.devotion` | Ceremonies and pilgrimages earn Devotion. Reach thirty. |
| `npc.progress` | You have broken {0} of the three wardens. The harvest grows stronger. |
| `npc.continue` | Continue |
| `npc.leave` | Leave |

### `act.*`

| Key | English value |
|---|---|
| `act.intro` | The road to {0} opens. {1} waits at its end. |
| `act.clear` | {0} falls. The road beyond is open. |
| `act.reward` | +{0} devotion |

### `hud.*`

| Key | English value |
|---|---|
| `hud.day` | Day {0} |
| `hud.fps` | FPS |
| `hud.floor` | FLOOR |
| `hud.resources` | Wood {0}     Stone {1}     Pearls {2}     Food {3} |
| `hud.companions` | Grace {0}     Companions {1}     Omens {2}     {3} |
| `hud.items` | Meals {0}/{1}     Herbs {2}/{3}     Medicine {4}/{5}     Potions {6}     Bombs {7}/{8}     Rite {9} [X]     [I] inventory |
| `hud.boss.unknown` | THE GILDED STAG |

`hud.resources`, `hud.companions` and `hud.items` are single-row layouts: their
column spacing lives in the value, so a translation has to keep the multi-space
gaps or the HUD rows misalign.

### `phase.*`

| Key | English value |
|---|---|
| `phase.dawn` | Dawn |
| `phase.day` | Day |
| `phase.dusk` | Dusk |
| `phase.night` | Night |

### `prompt.*`

| Key | English value |
|---|---|
| `prompt.oracle` | [E]  Talk to the Oracle |
| `prompt.mourn` | [E]  Mourn {0} |
| `prompt.remember` | [E]  Remember {0} |
| `prompt.farm` | [E]  Harvest {0} food |
| `prompt.farm.growing` | The rose garden is growing... |
| `prompt.lumber` | [E]  Collect {0} wood |
| `prompt.lumber.working` | The timber grove is working... |
| `prompt.quarry` | [E]  Collect {0} stone |
| `prompt.quarry.working` | The marble quarry is working... |
| `prompt.herbs` | [E]  Harvest {0} herbs |
| `prompt.herbs.growing` | The herb garden is growing... |
| `prompt.gather` | [E]  Gather {0}   ({1} left) |
| `prompt.altar` | [E]  Ceremonies |
| `prompt.altar.resting` | HarvestAltar resting ({0}s) |
| `prompt.portal` | [E]  Open the pilgrimage map |
| `prompt.default` | [B]  Build    •    [E]  Interact |
| `prompt.manage` | [R] manage |
| `prompt.leave.interior` | [E]  Leave the interior |
| `prompt.interior` | {0}   •   [E] door to leave |

### `building.*`

| Key | English value |
|---|---|
| `building.farm` | Rose Garden |
| `building.shrine` | Sunwell |
| `building.bed` | Silk Bed |
| `building.lumberyard` | Timber Grove |
| `building.quarry` | Marble Quarry |
| `building.kitchen` | Kitchen Pavilion |
| `building.storage` | Vault |
| `building.herbgarden` | Herb Terrace |
| `building.apothecary` | Elixir Atelier |
| `building.teahouse` | Tea House |
| `building.memorial` | Memory Garden |
| `building.reflection` | Reflection Chamber |
| `building.sunbath` | Sun Bath |
| `building.obelisk` | Obelisk |
| `building.temple` | Grand Temple |
| `building.sawmill` | Timber Mill |
| `building.stoneworks` | Stoneworks |
| `building.wheatfield` | Wheat Field |
| `building.belltower` | Bell Tower |
| `building.reliquary` | Reliquary |

### `room.*`

| Key | English value |
|---|---|
| `room.entrance` | Entrance |
| `room.combat` | Combat |
| `room.elite` | Elite |
| `room.treasure` | Treasure |
| `room.shop` | Shop |
| `room.curse` | Curse |
| `room.puzzle` | Rune Puzzle |
| `room.rescue` | Rescue |
| `room.forge` | Forge |
| `room.boss` | Warden |

!!! warning "`room.boss` is not the key the code asks for"
    `FloorPlanGenerator.Name` builds `room.warden` from `RoomType.Warden`, so this
    table's `room.boss` entry is never read. Either rename the key to `room.warden`
    or accept the English fallback for Warden rooms.

### `menu.*`

| Key | English value |
|---|---|
| `menu.build.title` | BUILD   •   PLACE ON THE GRID |
| `menu.build.hint` | Press 1-6   •   Esc to close |
| `menu.next` | Next page |
| `menu.previous` | Previous page |
| `menu.companions.title` | COMPANIONS |
| `menu.companions.hint` | Press 1-6   •   Esc to close |
| `menu.companions.empty` | No companions yet |
| `menu.actions.job` | Assign job |
| `menu.actions.feed` | Feed |
| `menu.actions.treat` | Treat |
| `menu.actions.duty` | Duty |
| `menu.actions.bed` | Bed / Rest |
| `menu.actions.reflection` | Reflection Chamber |
| `menu.actions.release` | Release |
| `menu.back` | Back |
| `menu.close` | Close |
| `menu.duty.work` | Work priority |
| `menu.duty.grace` | Grace priority |
| `menu.duty.social` | Social priority |
| `menu.duty.bless` | Bless |
| `menu.bed.title` | BED |
| `menu.assign.title` | ASSIGN *(duplicate, overwritten below)* |
| `menu.assign.unassign` | Unassign job |
| `menu.assign.none` | No free workplaces |
| `menu.storage.title` | STORES |
| `menu.kitchen.title` | KITCHEN   •   RECIPES |
| `menu.apothecary.title` | APOTHECARY   •   RECIPES |
| `menu.collect.none` | Nothing to collect |
| `menu.reflection.empty` | The reflection chamber is empty |
| `menu.bed.rest` | Rest here now |
| `menu.bed.free` | free |
| `menu.bed.slot` | Bed {0}   •   {1} |
| `menu.bed.hint` | Press a number   •   Esc to go back |
| `menu.assign.current` | current |
| `menu.assign.title` | ASSIGN {0} |
| `menu.assign.hint` | Press 1-6   •   Esc to go back |

### `companion.*`

| Key | English value |
|---|---|
| `companion.idle` | Idle |
| `companion.fed` | Fed |
| `companion.partner` | partner {0} |
| `companion.friend` | friend {0} |
| `companion.child` | child of {0} |
| `companion.feed.meal` | {0} eats a meal |
| `companion.feed.food` | {0} is fed |
| `companion.no.food` | No food |
| `companion.treat.none` | No medicine   •   assign a worker to an apothecary |
| `companion.treated` | {0} is treated |
| `companion.bless.cost` | Blessing costs 5 grace |
| `companion.blessed` | {0} feels the blessing |
| `companion.rest` | {0} goes to rest |
| `companion.claims` | {0} claims {1} |
| `companion.idle.toast` | {0} is now idle |
| `companion.works` | {0} works the {1} |
| `companion.reflection.releasefirst` | Release them from the reflection chamber first |

### `trait.*`

| Key | English value |
|---|---|
| `trait.loyal` | Loyal |
| `trait.hardworking` | Hardworking |
| `trait.glutton` | Glutton |
| `trait.devout` | Devout |
| `trait.sickly` | Sickly |
| `trait.lazy` | Lazy |
| `trait.loyal.desc` | loyalty fades slowly |
| `trait.hardworking.desc` | steadier worker, +2 output |
| `trait.glutton.desc` | hungers quickly |
| `trait.devout.desc` | +2 grace at hymns |
| `trait.sickly.desc` | falls ill easily |
| `trait.lazy.desc` | works slowly, tires quickly |

### `mood.*`

| Key | English value |
|---|---|
| `mood.dissenting` | Dissenting |
| `mood.ill` | Ill |
| `mood.starving` | Starving |
| `mood.exhausted` | Exhausted |
| `mood.sleeping` | Sleeping |
| `mood.hungry` | Hungry |
| `mood.tired` | Tired |
| `mood.devoted` | Devoted |
| `mood.content` | Content |

### `priority.*`

| Key | English value |
|---|---|
| `priority.work` | Work |
| `priority.grace` | Grace |
| `priority.social` | Social |

### `ceremony.*`

| Key | English value |
|---|---|
| `ceremony.title` | CEREMONIES |
| `ceremony.hymn` | Hymn   •   free   •   1 food per companion |
| `ceremony.feast` | Feast   •   {0} food   •   +20 grace, feeds everyone |
| `ceremony.sacrifice` | Sacrifice   •   1 companion   •   +45 grace, +10 pearls |
| `ceremony.sharpen` | Sharpen blade   •   15 pearls   •   +5 damage |
| `ceremony.plenty` | Ceremony of Plenty   •   10 food   •   +5 meals, +10 grace |
| `ceremony.warding` | Ceremony of Warding   •   10 pearls   •   cure and strengthen every companion |
| `ceremony.resting` | The altar is resting ({0}s) |
| `ceremony.sharpen.cost` | Sharpening the blade costs {0} pearls |
| `ceremony.sharpen.done` | The blade is sharpened   •   damage {0} |

### `daily.*`

| Key | English value |
|---|---|
| `daily.harvest` | A wild harvest arrives   •   +6 food |
| `daily.pilgrims` | Pilgrims leave an offering   •   +10 pearls |
| `daily.inspired` | The harvest wakes inspired   •   +8 grace |
| `daily.storm` | A storm batters the {0}   •   repair it |
| `daily.quiet` | A quiet dawn   •   +4 wood |
| `daily.fellowship` | A day of fellowship   •   +2 loyalty |
| `daily.altar` | The altar hums   •   +5 grace |

### `creed.*`

| Key | English value |
|---|---|
| `creed.none` | No creed |
| `creed.adopted` | {0} adopted |
| `creed.set` | The harvest is set in its ways   •   +25 grace |

### `dungeon.*`

| Key | English value |
|---|---|
| `dungeon.corridor` | Follow the corridor |
| `dungeon.clear` | Clear the room ({0}) |
| `dungeon.doorsopen` | Doors open |
| `dungeon.elite` | Defeat the elite ({0}) |
| `dungeon.chest` | Open the chest |
| `dungeon.treasuretaken` | Treasure taken |
| `dungeon.runes` | Runes {0}/{1} |
| `dungeon.trade` | Trade, then choose a door |
| `dungeon.altarwaits` | The altar waits |
| `dungeon.captivefree` | The captive is free |
| `dungeon.freecaptive` | Free the captive |
| `dungeon.forgecold` | The forge is cold |
| `dungeon.useforge` | Use the forge |
| `dungeon.bosschest` | Open the boss chest |
| `dungeon.slay` | Slay {0} |
| `dungeon.choosedoor` | Choose a door |
| `dungeon.prompt.descend` | [E]  Descend deeper |
| `dungeon.prompt.return` | [E]  Return to the Harvesthold |
| `dungeon.prompt.openchest` | [E]  Open the chest |
| `dungeon.prompt.freecaptive` | [E]  Free the captive |
| `dungeon.prompt.useforge` | [E]  Use the forge |
| `dungeon.prompt.trade` | [E]  Trade with the merchant |
| `dungeon.prompt.altar` | [E]  Touch the cursed altar |
| `dungeon.prompt.runes` | Step on the runes   •   {0}/{1} |
| `dungeon.prompt.unlock` | [E]  Unlock {0}   •   keys {1} |
| `dungeon.prompt.travel` | [E]  Travel to {0} |
| `dungeon.locked` | The door is locked   •   find a key |
| `dungeon.unlocked` | Door unlocked   •   keys {0} |
| `dungeon.corridor.toast` | Corridor   •   follow the path |
| `dungeon.captive.cap` | The harvest is at its companion cap |
| `dungeon.captive.joined` | A captive joins the harvest |

### `shop.*`

| Key | English value |
|---|---|
| `shop.title` | MERCHANT |
| `shop.heal` | Heal 30   •   15 pearls   •   stock {0} |
| `shop.key` | Buy a key   •   10 pearls   •   stock {0} |
| `shop.heirloom` | {0}   •   25 pearls   •   stock {1} |
| `shop.potion` | Healing potion   •   18 pearls   •   stock {0} |
| `shop.attire` | {0}   •   30 pearls   •   stock {1} |
| `shop.sell` | Sell goods   •   trade surplus for pearls |
| `shop.hint` | Press 1-6   •   Esc to close |
| `shop.soldout` | The merchant is sold out |
| `shop.notenoughpearls` | Not enough pearls |
| `shop.noheirloom` | The merchant has no heirloom to sell |
| `shop.heirloomcost` | A heirloom costs 25 pearls |
| `shop.noattire` | The merchant has no attire to sell |
| `shop.attirecost` | That attire costs 30 pearls |
| `shop.potioncost` | A potion costs 18 pearls |
| `shop.sell.title` | MERCHANT   •   SELL |
| `shop.sell.wood` | Sell 10 wood   •   +8 pearls |
| `shop.sell.stone` | Sell 10 stone   •   +8 pearls |
| `shop.sell.food` | Sell 5 food   •   +6 pearls |
| `shop.sell.herbs` | Sell 2 herbs   •   +5 pearls |
| `shop.sell.back` | Back to buying |
| `shop.sell.none` | Not enough goods to sell |

### `forge.*` and `curse.*`

| Key | English value |
|---|---|
| `forge.title` | FORGE |
| `forge.sharpen` | Sharpen blade   •   15 pearls   •   +5 damage |
| `forge.leave` | Leave the forge |
| `curse.title` | OMEN ALTAR |
| `curse.take` | Take the gift   •   +1 key, +25 pearls, -20 health |
| `curse.refuse` | Refuse |

### `recipe.*`

| Key | English value |
|---|---|
| `recipe.kitchen.one` | Cook a meal   •   2 food   •   +1 meal |
| `recipe.kitchen.three` | Cook three meals   •   5 food + 1 herb   •   +3 meals |
| `recipe.kitchen.feast` | Cook a feast   •   7 food   •   +3 meals and +6 grace |
| `recipe.kitchen.two` | Cook two meals   •   4 food   •   +2 meals |
| `recipe.kitchen.hearty` | Cook a hearty feast   •   8 food + 1 herb   •   +4 meals and +8 grace |
| `recipe.apothecary.medicine` | Brew medicine   •   2 herbs   •   +1 medicine |
| `recipe.apothecary.potion` | Brew healing potion   •   2 herbs + 1 medicine   •   +1 potion |
| `recipe.apothecary.bomb` | Craft a bomb   •   1 herb + 2 stone   •   +1 bomb |
| `recipe.apothecary.potions` | Brew two potions   •   3 herbs + 1 medicine   •   +2 potions |
| `recipe.apothecary.bombs` | Craft two bombs   •   2 herbs + 4 stone   •   +2 bombs |

### `toast.*`

| Key | English value |
|---|---|
| `toast.health.full` | Health is already full |
| `toast.potion.none` | No healing potions |
| `toast.potion.used` | Potion used   •   +40 health |
| `toast.shrine` | The sunwell gathers grace on its own |
| `toast.kitchen` | The kitchen is ready   •   feasts cost {0} food |
| `toast.storage` | Storage   •   caps raised to {0} wood / {1} food |
| `toast.apothecary` | Apothecary   •   {0} medicine   •   [R] manage |
| `toast.bed` | A companion's bed |
| `toast.interior` | Interior   •   {0} |
| `toast.backharvest` | Back in the harvest |
| `toast.leave.door` | Leave through the door |

### `pause.*`

| Key | English value |
|---|---|
| `pause.title` | PAUSED |
| `pause.resume` | Resume |
| `pause.save` | Save game |
| `pause.load` | Load game |
| `pause.settings` | Preferences |
| `pause.photo` | Photo mode |
| `pause.quit` | Quit to title |

### `settings.*`

| Key | English value |
|---|---|
| `settings.title` | SETTINGS |
| `settings.master` | Master volume |
| `settings.music` | Music volume |
| `settings.sfx` | SFX volume |
| `settings.shake` | Screen shake |
| `settings.reduced` | Reduced motion |
| `settings.more` | More options |
| `settings.uiscale` | UI scale |
| `settings.fullscreen` | Fullscreen |
| `settings.fps` | FPS counter |
| `settings.autosave` | Autosave |
| `settings.rebind` | Rebind keys |
| `settings.back` | Back |
| `settings.on` | On |
| `settings.off` | Off |

### `slots.*`

| Key | English value |
|---|---|
| `slots.title` | SAVE SLOTS |
| `slots.slot` | Slot {0} |
| `slots.auto` | Autosave |
| `slots.empty` | empty |
| `slots.saved` | Saved to slot {0} |
| `slots.loaded` | Loaded slot {0} |
| `slots.missing` | That slot is empty |
| `slots.back` | Back |
| `slots.delete` | Delete a slot |
| `slots.delete.title` | DELETE A SLOT |
| `slots.deleted` | Slot deleted |
| `slots.overwrite.title` | OVERWRITE SAVE? |
| `slots.overwrite` | Overwrite this slot |
| `slots.cancel` | Cancel |

### `rebind.*` and `photo.*`

| Key | English value |
|---|---|
| `rebind.title` | REBIND KEYS |
| `rebind.attack` | Attack |
| `rebind.dodge` | Dodge |
| `rebind.interact` | Interact |
| `rebind.prompt` | Press a key   •   Esc cancels |
| `photo.prompt` | PHOTO MODE   •   WASD pan   •   Scroll zoom   •   F12 capture   •   Esc exit |

## The pseudo-locale

`xx.json` is a layout test, not a translation: it has the same 349 keys in the same
order, and every value is the English text prefixed with the literal ASCII marker
`?XX? `.

```json
{
  "entries": [
    { "key": "title.name", "value": "?XX? CHOSEN OF THE HARVEST" },
    { "key": "dungeon.runes", "value": "?XX? Runes {0}/{1}" }
  ]
}
```

Because the marker is plain ASCII it renders in any font, so a missing glyph shows
up immediately, and `{0}` placeholders survive so `Format` calls still exercise
their arguments. Two ways to use it:

- **In the build:** Preferences → page 2 → `Language` switches English ⇄
  `Pseudo (layout)` at runtime.
- **In the smoke test:** `SmokeTest` switches to `xx`, formats `hud.day`, and logs
  `SMOKE locale pseudo=… keys=349` before switching back — the fastest way to catch
  a table that failed to load.

!!! tip "Add a real locale"
    1. Copy `en.json` to `<code>.json` (for example `de.json`) in
       `Resources/Localization`.
    2. Keep every `key` string byte-identical and keep the `{0}`–`{3}`
       placeholders in the translated values.
    3. Keep layout-bearing spacing in `hud.*` and the `  •  ` separators.
    4. Switch with `Lexicon.SetLocale("de")`, then confirm `Lexicon.KeyCount`
       matches 349.
    5. If the table fails to parse, `Lexicon` logs
       `Lexicon table failed to load: …` and falls back to code defaults.

## Related

- [Localization guide](../guide/localization.md) — the workflow around these tables
- [Followers API](followers.md) — mood, trait and priority keys in context
- [Dungeon API](dungeon.md) — `dungeon.*` objective and prompt keys
- [Rebranding the template](../customization/rebranding.md) — renaming the lore in every table

