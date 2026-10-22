# Chosen of the Harvest

**A complete, code-driven action-roguelike + sanctuary-management framework for
Unity 6 â€” ready to reskin and ship.**

Chosen of the Harvest is a production-ready Unity template: you build a
sanctuary, recruit and manage a cast of followers, then descend into repeatable
expeditions to break the Wardens who hoard the harvest. Every system is written
in C# and generated at runtime, so there is no prefab wiring to maintain and no
scene to repair â€” reskin the art, retune the numbers, and it is your game.

<div class="grid cards" markdown>

- :material-rocket-launch: **Runs out of the box**

    ---

    Open the project, press **Play**. The hub, the followers, the HUD, the
    dungeon and the bosses assemble themselves from code.

    [:octicons-arrow-right-24: Your first run](getting-started/first-run.md)

- :material-palette: **Built to reskin**

    ---

    All presentation flows through catalogs and `Assets/ChosenOfTheHarvest/Resources`,
    so replacing art, audio and copy never touches gameplay logic.

    [:octicons-arrow-right-24: Customization](customization/rebranding.md)

- :material-cube-outline: **Systems, not a demo**

    ---

    Save slots with migration and backup recovery, procedural floors, a follower
    simulation with needs and jobs, quests, meta-progression and localisation.

    [:octicons-arrow-right-24: Architecture](architecture/overview.md)

- :material-bookshelf: **Documented**

    ---

    Every system has a guide page, and the public API surface is catalogued in
    the reference section.

    [:octicons-arrow-right-24: Reference](reference/core.md)

</div>

![The Harvesthold hub](assets/images/hub.jpg)

## What you get

| Area | Included |
|---|---|
| **Unity version** | 6000.6.6f1 (Unity 6), Universal Render Pipeline |
| **Render / input** | URP + 2D renderer, Unity Input System |
| **Code** | 39 C# files, ~13.7k lines, single `ChosenOfTheHarvest` namespace |
| **Scenes** | One bootstrap scene â€” everything else is generated |
| **Player** | Movement, dodge with i-frames, 12 armaments, 3 rites, bombs, inventory |
| **Combat** | 12 creature archetypes, 3 elite modifiers, telegraphed attacks, projectiles, traps |
| **Bosses** | 3 Wardens with unique movesets, arenas, phases and Ascendant forms |
| **Dungeon** | Procedural floor plans, 3 acts / 6 floors / 92 rooms, keys, shops, puzzles |
| **Sanctuary** | 20 sanctum types, levels, condition, dirt, workers, player-built interiors |
| **Followers** | Needs, traits, loyalty, sickness, relationships, children, jobs, priorities |
| **Progression** | 18 omens, 18 heirlooms, 11 attire pieces, devotion meta-upgrades, 9-stage quest chain |
| **Persistence** | Save schema 4 â€” 3 slots + autosave, version migration, `.bak` recovery |
| **Presentation** | Fully code-generated HUD, photo mode, settings, key rebinding, mute |
| **Text** | 349 localisation keys, English + pseudo-locale, runtime hot switch |
| **Content** | 49 placeholder sprites + 21 placeholder audio clips, all replaceable |

## Requirements

!!! note "Before you start"

    - **Unity 6000.6.6f1** or newer Unity 6 release (earlier Unity 6 releases
      will prompt an upgrade).
    - **Input System** and **Universal Render Pipeline** packages â€” both are
      already declared in `Packages/manifest.json`.
    - No third-party assets, plugins or paid dependencies. Everything included
      is generated in-house and covered by the template licence.

## Pick a path

<div class="grid cards" markdown>

- :material-download: **I want to run it**

    ---

    [Installation](getting-started/installation.md) â†’
    [Your first run](getting-started/first-run.md) â†’
    [Controls](getting-started/controls.md)

- :material-brush: **I want to reskin it**

    ---

    [Replacing art](customization/art.md) â†’
    [Replacing audio](customization/audio.md) â†’
    [Rebranding](customization/rebranding.md)

- :material-tune: **I want to change the game**

    ---

    [Balancing & tuning](customization/balancing.md) â†’
    [Adding content](customization/adding-content.md) â†’
    [Systems map](architecture/systems-map.md)

- :material-publish: **I want to ship it**

    ---

    [Building players](publishing/building.md) â†’
    [Asset Store submission](publishing/asset-store.md) â†’
    [License & credits](license.md)

</div>

## Support

- Browse the [FAQ](faq.md) and [Troubleshooting](troubleshooting.md) pages first.
- Check the [Changelog](changelog.md) for what changed in each release.
- Still stuck? Open an issue on
  [GitHub](https://github.com/camenduru/ChosenOfTheHarvestDoc/issues) with your
  Unity version, platform and the relevant log excerpt.
