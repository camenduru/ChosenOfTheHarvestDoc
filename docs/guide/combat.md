# Combat in the Fallow

Combat in *Chosen of the Harvest* is built from four verbs — swing, dodge, cast
and throw — layered over a fixed list of armaments, rites and husk archetypes.
Every number below comes from the same handful of classes, so the feel of a run
is the sum of your loadout, your omens and heirlooms, and the act you chose. This
page covers the controls, the damage formulas, every creature family, and the
three wardens waiting at the end of each act.

!!! tip "The four verbs"
    ++space++ attacks with the equipped armament, ++left-shift++ rolls, ++x++
    casts the equipped rite and ++g++ throws a bomb. All four are rebindable in
    settings; the defaults live in `Bindings.Defaults` (`Scripts/Core/Controls.cs`).

## Controls and the shape of a fight

A fight is deliberately readable. Husks telegraph with a red ground ring before
they strike, your own swings lunge you a short distance forward, and the dodge
roll grants more invulnerability than it consumes in time. The result is a
loop of commit, read, roll, punish.

| Input | Default | Behaviour |
|---|---|---|
| Attack | ++space++ | Swings the current armament; combos chain inside `ComboWindow` |
| Dodge | ++left-shift++ | 0.22 s roll at 11 m/s, 0.42 s of i-frames, 0.75 s cooldown |
| Rite | ++x++ | Radial burst centred on the player |
| Bomb | ++g++ | Lobs a bomb 3.4 m ahead; 0.9 s cooldown, consumes one bomb |
| Potion | ++h++ | Drinks a potion from the inventory |
| Interact | ++e++ | Doors, chests, altars, the portal |

The player's own hurtbox is generous in your favour: after taking a hit you get
0.7 s of invulnerability, so overlapping husks cannot chain-stun you to death.

## Armaments

Twelve armaments are defined in `ArmamentCatalog`. They are not sidegrades with
different skins — each changes range, arc, knockback, combo count and status
application. `HarvestState.Data.weaponType` stores the equipped id, and the
armory in the hub cycles through them.

| Armament | Damage | Cooldown | Range | Combo | Notable property |
|---|---|---|---|---|---|
| Rose Saber | 25 | 0.34 s | 1.9 | 3 | Baseline; balanced arc |
| Golden Axe | 42 | 0.58 s | 2.05 | 1 | Slow 0.55× for 1.3 s, heavy knockback |
| Fan Blade | 13 | 0.19 s | 1.45 | 4 | Bleed 3 dps for 3.5 s |
| Silk Lance | 30 | 0.42 s | 2.65 | 2 | Longest reach of the light weapons |
| Bell Hammer | 52 | 0.88 s | 2.15 | 1 | Hits all around; slow 0.45× for 1.6 s |
| Thorn Claws | 9 | 0.14 s | 1.25 | 5 | 1 lifesteal per hit |
| Crescent Scythe | 36 | 0.5 s | 2.5 | 2 | Bleed 2 dps for 3 s, wide arc |
| Sunburst Mace | 58 | 1.0 s | 2.0 | 1 | Hits all; slow 0.4× for 2 s |
| Chain of Petals | 38 | 0.72 s | 2.35 | 2 | Hits all; slow 0.5× for 1.4 s |
| Glass Rapier | 20 | 0.22 s | 2.2 | 4 | Long lunge (0.42), narrow arc |
| Jade Cleaver | 46 | 0.66 s | 1.85 | 2 | Bleed 5 dps for 4 s |
| Royal Halberd | 40 | 0.55 s | 2.8 | 2 | Longest reach overall |

`HitsAll` weapons skip the facing check entirely — the Bell Hammer, Sunburst
Mace and Chain of Petals hit every husk inside `Range`. Everything else compares
`Vector3.Dot(facingDir, direction)` against the weapon's `ArcDot`, so a negative
value like the axe's `-0.05` still covers a wide cone.

!!! note "The combo bonus is per swing, not per enemy"
    Combo index increments once per attack and resets after `ComboWindow`
    elapses. Each step adds 15% damage to every husk hit by that swing.

## Damage, in full

`PlayerController.Attack()` builds the final number from the equipped armament,
your progression and your relics:

```csharp
bool critical = UnityEngine.Random.value < HarvestState.CriticalChance;
int damage = Mathf.RoundToInt(HarvestState.WeaponDamage * (1f + comboIndex * 0.15f) *
    (critical ? HarvestState.CriticalMultiplier : 1f));
```

`HarvestState.WeaponDamage` is the other half of the story:

```csharp
int damage = ArmamentCatalog.Current.BaseDamage + BonusDamage + Data.weaponLevel * 5 +
             (HasOmen("star") ? 5 : 0) + (HasOmen("emperor") ? 2 : 0) +
             Data.metaWrath * 4 + (HasHeirloom("fang") ? 5 : 0);
float multiplier = (Data.charmId == "ember" ? 1.12f : 1f) *
                   (HasOmen("death") ? 1.1f : 1f);
return Mathf.RoundToInt(damage * multiplier);
```

Criticals start at 8% and 1.6×, and several items move both figures. On the
receiving end, `Creature.Hit` applies flat reduction first (the Mirror Sentinel
has 20%, Armored elites 30%), and `PlayerController.TakeDamage` applies your
own mitigation last:

```csharp
damage = Mathf.Max(1, Mathf.RoundToInt(
    damage * HarvestState.DamageTakenMultiplier * Preferences.IncomingDamageMultiplier));
```

The `Mathf.Max(1, ...)` is load-bearing: no stack of armour can make you
literally immune to a hit.

## Rites

Rites are radial, instant, and independent of your armament. All three are
unlocked from the start so the combat sandbox can be tested end to end, and the
ritual menu cycles between them.

| Rite | Damage | Radius | Cooldown | Extra effect |
|---|---|---|---|---|
| Sunburst | 34 | 3.5 | 7.5 s | Strong knockback (4.5) |
| Shroud of Frost | 18 | 4.1 | 8.5 s | Slow 0.42× for 2.8 s |
| Crimson Vow | 22 | 3.2 | 9.5 s | Heals 5 per husk hit |

Rite damage adds a third of your Wrath meta upgrade, then multiplies:

```csharp
int damage = Mathf.Max(1, Mathf.RoundToInt(
    (rite.Damage + HarvestState.Data.metaWrath * 3) * HarvestState.RiteDamageMultiplier));
```

Cooldowns are multiplied by `RiteCooldownMultiplier`, which the Tide charm
(0.8×), Moon armour (0.9×), the Judgement omen (0.75×) and the Bell heirloom
(0.9×) all reduce. A fully stacked rite build casts roughly twice as often as a
bare one. Hits are counted for the floating text and for Crimson Vow's healing,
so a rite into a packed room is worth far more than its listed damage.

## Bombs

Bombs are consumables, not a cooldown ability. `ThrowBomb` refuses to fire at
zero stock and points you at the apothecary; otherwise it spends one, waits
0.9 s, and lobs the bomb at a point 3.4 m ahead.

| Property | Value |
|---|---|
| Direct damage | 30 + `metaWrath` × 2 |
| Blast radius | 2.4 |
| Burn | 3 s at 4 dps |
| Knockback | 5 |
| Cooldown | 0.9 s |

Bombs ignore the facing cone, so they are the answer to a room that has already
surrounded you. The burn ticks every half second like every other damage-over-time
effect.

## Status effects

`Creature` tracks four timed effects and ticks them all through one method.
Applications take the stronger value rather than stacking, so re-applying a
weaker bleed never overwrites a stronger one.

| Status | Applied by | Tick |
|---|---|---|
| Bleed | Fan Blade, Crescent Scythe, Jade Cleaver | 0.5 s, `ceil(dps × 0.5)` |
| Burn | Bombs | 0.5 s |
| Poison | Cinder husks and dungeon hazards | 0.5 s |
| Slow | Axe, hammer, mace, flail, Shroud of Frost | Continuous factor, clamped 0.25–1 |

```csharp
public void ApplyBleed(float duration, float dps)
{
    if (dead) return;
    bleedTime = Mathf.Max(bleedTime, duration);
    bleedDps = Mathf.Max(bleedDps, dps);
}
```

Because duration and dps are each maxed independently, a weak long bleed and a
strong short bleed combine into a strong long one. That is intentional: it makes
mixed parties feel better than the sum of their parts.

## Creature archetypes

Every husk is a `Creature` with an `Archetype`. The archetype sets speed, attack
range and attack cooldown at spawn, and chooses which brain runs in `Update`.
The twelve families are:

| Archetype | In-game name | Speed | Range | Cooldown | Behaviour |
|---|---|---|---|---|---|
| Melee | Thornbound Duelist | 2.3 | 1.05 | 1.1 s | Balanced chaser |
| Fast | Silk Wisp | 3.15 | 0.95 | 0.85 s | Fragile rusher |
| Brute | Rootbound Ogre | 1.65 | 1.35 | 1.45 s | Slow heavy hitter |
| Caster | Moth Seer | 2.0 | 1.05 | 1.3 s | Keeps 3.1–6.4 m, fires bolts |
| Charger | Briar Hound | 2.5 | 1.0 | 1.2 s | Wind-up then long dash |
| Exploder | Cinder Husk | 2.9 | 0.9 | 1.0 s | Detonates on death |
| Shielder | Mirror Sentinel | 1.5 | 1.2 | 1.7 s | 20% damage reduction |
| Summoner | Fallow Choir | 2.1 | 1.05 | 1.6 s | Summons capped minions |
| Healer | Verdant Mender | 2.3 | 1.05 | 1.5 s | Heals nearby husks |
| Archer | Glass Archer | 2.5 | 1.05 | 1.5 s | Long-range rapid bolts |
| Slammer | Bell Golem | 1.6 | 1.7 | 2.1 s | Telegraphed radial shockwave |
| Assassin | Shroud Stalker | 3.35 | 1.0 | 0.9 s | Blinks behind the Sovereign |

The brains differ in more than stats. The Archer fires every 1.7 s and keeps a
4.5–8.5 m band; the Caster fires every 2.5 s and prefers 3.1–6.4 m. The
Summoner retreats below 4.2 m and advances past 7 m, summoning on a 4–7 s timer.
The Healer holds 4–8 m and heals allies on a 3.5–6 s timer. The Slammer winds up
for 0.55 s and deals 1.2× damage in a 2.8 m radius. The Assassin blinks to a
random point within 1.6 m of you and resets its attack timer on arrival.

!!! warning "Elites are modifiers, not archetypes"
    An elite keeps its base brain and adds one of three modifiers:
    **Armored** (30% reduction), **Frenzied** (+25% speed, ×0.8 cooldown) or
    **Vampiric** (heals 30% of damage dealt). A Vampiric Brute is a very
    different problem from a Vampiric Wisp.

## The three wardens

Each act ends in a `WardenController` layered over a regular `Creature`. The
warden flips to phase two at 50% health, shortening its telegraphs and adding
radial volleys. Arenas are dressed by `WardenArena` but share the same collision
and room graph.

| Act | Warden | Arena theme | Moveset |
|---|---|---|---|
| 0 | The Gilded Stag | Gilded Court | Antler charge, antler volley, slam |
| 1 | The Mire Queen | Drowned Ring | Thorn snare, mire fan, slam |
| 2 | The Obsidian Sultan | Obsidian Court | Mirage surge, teleport, radial bolts |

The shared slam is the backbone of all three fights:

```csharp
float radius = phase == 1 ? 2.8f : 3.5f;
float duration = phase == 1 ? 0.8f : 0.55f;
...
pc.TakeDamage(phase == 1 ? 18 : 24);
```

Phase two also changes the follow-up. The Stag fires 8 radial bolts (11 damage,
4.4 m/s); the Mire Queen fires 11 green bolts and a 5-bolt fan; the Sultan fires
14 blue bolts and teleports next to you. Attack timers shorten from 3.0 s to
2.0 s for acts 0 and 1, and from 2.4 s to 1.5 s for act 2 — the Sultan is
relentless by design.

!!! tip "Roll through, not away"
    Every warden attack is a ground-centred radius or a slow projectile. The
    dodge's 0.42 s of i-frames is longer than the 0.55 s phase-two slam
    telegraph only if you start rolling late. Rolling early leaves you standing
    in the ring when it lands.

## Difficulty

Difficulty is a persisted preference, not a per-run choice. It scales incoming
damage and creature health, and nothing else — enemy damage, speed and loot stay
identical.

| Setting | Incoming damage | Creature health |
|---|---|---|
| Easy | 0.7× | 0.85× |
| Normal | 1.0× | 1.0× |
| Hard | 1.3× | 1.2× |

```csharp
public static float IncomingDamageMultiplier => Difficulty switch
{
    0 => 0.7f,
    2 => 1.3f,
    _ => 1f
};
```

## Under the hood

Combat lives in three files. `PlayerController` owns the four verbs, their
cooldowns and the hurtbox. `Creature` owns every husk's brain, status timers and
death. `WardenController` owns the two-phase boss script.

`Creature.Init` is where an archetype becomes a stat block. It reads
`Preferences.CreatureHealthMultiplier` once, so difficulty is baked in at spawn
rather than applied per hit:

```csharp
MaxHealth = Mathf.Max(1, Mathf.RoundToInt(health * Preferences.CreatureHealthMultiplier));
Health = MaxHealth;
Damage = Mathf.Max(1, damage);
IsWarden = boss;
Kind = kind;
Phase = 1;
```

The brains are plain distance switches in `Update`. There is no behaviour tree,
no navmesh and no steering beyond `Move(direction)`, which resolves against
`RoomCollision` and slides along walls when blocked. That is why husks can be
baited around pillars reliably — the slide is geometric, not pathfound.

Damage flows one way: `PlayerController.Attack` calls `Creature.Hit`, which
applies reduction and calls `Die()` at zero. `Die()` is also where the Exploder's
detonation, the loot roll and the room-clear check happen, so killing a Cinder
Husk in a doorway can still hurt you after the room is technically empty.

`Creature.All` is a static list maintained in `OnEnable`/`OnDisable`. Every
ability iterates a snapshot (`.ToArray()`) because a husk can die mid-loop and
remove itself. If you add a new area attack, do the same or you will hit
`InvalidOperationException` on the first kill.

The dodge is worth reading in full, because its numbers are the whole defensive
game:

```csharp
dodgeDir = moveInput.sqrMagnitude > 0.01f ? moveInput.normalized : facingDir;
dodgeTimer = 0.22f;
dodgeCooldown = 0.75f;
invuln = Mathf.Max(invuln, 0.42f);
```

`invuln` is shared with the post-hit grace period, and `Mathf.Max` means a dodge
can extend existing i-frames but never shorten them. Chaining a dodge directly
after a hit therefore gives you 0.42 s from the moment of the roll, not 0.7 s
plus 0.42 s.

## Related

- [Expeditions into the Fallow](expeditions.md) — how a run is generated and paced.
- [The Harvesthold hub and companions](hub-and-companions.md) — where loadouts are changed.
- [Economy and production](economy.md) — bombs, potions and the supplies that feed them.
- [Ceremonies, omens and heirlooms](ceremonies.md) — the buffs that reshape these numbers.
