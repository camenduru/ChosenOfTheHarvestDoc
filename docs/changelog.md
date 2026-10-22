# Changelog

All notable changes to the Chosen of the Harvest template are documented here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and
the package uses [Semantic Versioning](https://semver.org/).

## [1.0.0] — 2026-10-22

First public release.

### Added

- **Code-driven architecture.** One scene, one entry point. `HarvestGame` builds
  the camera, hub, player, companions, HUD and audio at runtime; there are no
  prefabs or scene references.
- **Sanctuary management.** 20 sanctum types with levels, condition, dirt,
  workers, production, repairs and player-built interiors (walls, doors, decor).
- **Follower simulation.** Needs, hunger, loyalty, exhaustion, sickness, traits,
  jobs, priorities, beds, relationships, children, memorials and death.
- **Expeditions.** Procedural floor plans across 3 acts / 6 floors / 92 rooms,
  with keys, shops, treasure, puzzles, curses and corridors.
- **Combat.** 12 armaments, 3 rites, bombs, dodge with i-frames,
  bleed/slow/burn/poison, 12 creature archetypes and 3 elite modifiers.
- **Three Wardens** with unique arenas, phases, Ascendant forms and deaths.
- **Progression.** 18 omens, 18 heirlooms, 11 attire pieces, devotion
  meta-upgrades and a 9-stage quest chain.
- **Persistence.** Save schema 4, three manual slots plus autosave, atomic
  writes, `.bak` recovery and a migration/repair pass.
- **Presentation.** Code-built uGUI HUD, photo mode, settings (volume,
  difficulty, UI scale, high contrast, reduced motion, FPS counter, autosave),
  key rebinding and mute.
- **Localisation.** 349 keys, English plus a pseudo-locale for layout testing,
  with runtime hot switching.
- **Content.** 49 placeholder sprites and 21 placeholder audio clips, all loaded
  by path and replaceable without code changes.
- **Editor tooling.** Scene builder, standalone player builders for Windows,
  Linux and macOS, and an `AssetPostprocessor` that enforces sprite import rules.
- **Verification.** A `-smoke` pass that exercises the whole game loop, plus
  `Tools/build.ps1 -RunSmoke`.
- **Documentation.** Offline manual in `Assets/ChosenOfTheHarvest/Documentation/`
  and this online site.
- **Compliance files.** `Third-Party Notices.txt` and `AI-Disclosure.md`.

### Changed

- Editor menu moved under **Tools → Chosen of the Harvest** to match Asset Store
  packaging rules.
- Build logs are written to `BuildLogs/` only; the player output folder contains
  no text files.

### Fixed

- Static runtime state is reset on subsystem registration, so the template
  supports Fast Enter Playmode with Domain Reload disabled (Unity 6.6+).
- `Time.timeScale` is now derived every frame from the pause, photo-mode and
  hit-stop state, so an overlapping hit-stop can no longer leave the game
  frozen.
- Replaced deprecated `FindObjectsByType(FindObjectsSortMode)` and
  `FindFirstObjectByType` calls with their current Unity 6 equivalents, removing
  all obsolete-API compiler warnings.
- Expeditions and interiors now pin a dark camera backdrop, so the hub's
  day-cycle tint can no longer show as a bright wedge outside room floors.
- The title screen subtitle no longer reads "A CLEAN-ROOM STUDY BUILD".

### Known limitations

- Rendering is 2D sprites in a 3D stage; there are no 3D character models.
- Combat is keyboard/gamepad only — there is no click-to-move.
- The catalogs are plain C# arrays, not `ScriptableObject` assets.
- Art and audio are placeholders; expect to replace them before shipping.
