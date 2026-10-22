# FAQ

## General

### Is this a finished game?

No. It is a **template**: a complete, playable framework you reskin and extend.
The included art and audio are placeholders, and the lore names are deliberately
generic so you can replace them.

### Which Unity version do I need?

**6000.6.6f1** (Unity 6) or a newer Unity 6 release. Unity 2022.3 and earlier are
not supported.

### Does it need URP?

Yes. The project uses the Universal Render Pipeline with the 2D renderer. HDRP is
not supported out of the box.

### Are there prefabs I can edit?

No. The template generates everything at runtime from code, which is why it
cannot break from missing references. If you prefer prefabs, the catalogs are
plain serializable classes and can be moved to `ScriptableObject`s.

### Can I use this in a commercial game?

Yes. The package is distributed under the Unity Asset Store EULA. You can ship
games built with it. You cannot resell or redistribute the template itself.

### Is the art AI-generated?

The 49 sprites are AI-generated and disclosed in
[AI disclosure](https://camenduru.github.io/ChosenOfTheHarvestDoc/license/#ai-disclosure).
The 21 audio clips are synthesised procedurally by the project's own code.

### Why is there no music track I recognise?

There is no third-party audio. Every clip is original.

## Setup

### The scene is empty when I press Play

You opened the wrong scene. Open
`Assets/ChosenOfTheHarvest/Scenes/ChosenOfTheHarvest.unity`. If that file is
missing, run **Tools → Chosen of the Harvest → Create or Repair Bootstrap Scene**.

### Everything is pink

The URP pipeline asset is not assigned. Open **Project Settings → Graphics** and
set `Assets/ChosenOfTheHarvest/Settings/UniversalRP.asset` as the default
pipeline asset, and the same file in **Quality**.

### `error CS0234: The type or namespace name 'InputSystem' does not exist`

The Input System package did not resolve. Open **Window → Package Manager → In
Project** and install/reinstall **Input System**, then let Unity restart.

### Can I open it in Unity 6000.5?

It may work, but the package targets 6000.6.6f1 and is tested there. Downgrades
are not supported by Unity.

### Does it work with Domain Reload disabled?

Yes. Fast Enter Playmode is supported: every static field is reset on subsystem
registration. See [Architecture](architecture/overview.md).

## Gameplay

### How do I get more companions?

Rescue them from the Fallow, or recruit from the generated pool. The cap is 30.
Beds raise the comfortable population.

### My companions keep starving

Build a **Rose Garden** and assign a worker, or hold a **Hymn** (it feeds one
food per companion). A **Kitchen Pavilion** lowers the feast cost from 8 to 4
food.

### How do I cure a sick companion?

Build an **Elixir Atelier** to make medicine, or hold the **Ceremony of
Warding** (10 pearls) to cure everyone at once.

### Where do pearls come from?

Combat kills, chests, boss rewards and the Sacrifice ceremony.

### How do I unlock Act II?

Defeat the Act I Warden, **The Gilded Stag**, in the Ashen Orchard.

### How do I get devotion?

Quests (2–4 per stage), bosses (8/12/16), pilgrimages, and the Sovereign's
Diadem heirloom.

### The build menu is missing a building

Some buildings unlock by campaign progress or require resources you do not have.
Check the cost line — the menu greys out unaffordable entries.

### Can I play with a controller?

Yes. See [Controls](getting-started/controls.md#gamepad).

## Customisation

### How do I rename the game?

**Project Settings → Player → Product Name** and **Company Name**, then update
`HarvestGame.Version` and the `en.json` title keys. See
[Rebranding the template](customization/rebranding.md).

### How do I replace the player sprite?

Replace `Resources/Art/Characters/sovereign.png` and keep the import settings
(400 px/unit for characters). See [Replacing art](customization/art.md).

### Can I add my own weapons?

Yes — add an `ArmamentDef` to `ArmamentCatalog.All`. See
[Adding content](customization/adding-content.md).

### Does changing a weapon id break saves?

Yes. Saves store ids. Add entries freely; renaming or removing needs a migration.

## Performance

### What frame rate should I expect?

The game is 2D sprites in URP with a code-generated uGUI HUD. It runs comfortably
at 60 fps on integrated graphics at 1280×720, which is the default window size.

### Why does the first frame hitch?

`HarvestGame` builds the hub, camera, HUD and audio on start. The generated audio
fallback clips are the most expensive part; shipping with the WAV files present
avoids synthesis.

### Can I reduce the build size?

Remove `Resources/Audio` if you will synthesise audio at runtime, and strip
unused engine modules from `Packages/manifest.json`.

## Support

### Where do I report a bug?

<https://github.com/camenduru/ChosenOfTheHarvestDoc/issues> — include your Unity
version, platform, and the relevant log excerpt.

### Where is the changelog?

[Changelog](changelog.md).

### Is there a Discord?

No. Use the issue tracker so answers stay searchable.
