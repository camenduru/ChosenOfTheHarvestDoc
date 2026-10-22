# Replacing art

Every visible sprite in the template is a loose PNG under
`Assets/ChosenOfTheHarvest/Resources/Art`, loaded by path at runtime. There are no
atlases, sprite sheets, prefabs or material variants to keep in sync — replace a
file, keep its name, and the game picks it up. The only contract is the import
settings applied by `ArtImporter`, which this page documents in full.

## How sprites are loaded

`GameBootstrap` calls `Resources.Load<Sprite>("Art/<folder>/<name>")` and builds a
camera-facing billboard with `Stage3D`. Six folders, 49 files:

| Folder | Files | Loaded by |
| --- | --- | --- |
| `Characters/` | 12 | player, companions, NPCs, every creature |
| `Sanctums/` | 22 | the 20 sanctum types, plus `chest` and `door` |
| `World/` | 5 | `ground`, `altar`, `statue`, `tree`, `rock` |
| `Expedition/` | 5 | floors, candle, projectile, skull |
| `Wardens/` | 4 | three boss bodies, plus unused `stag_antlers` |
| `UI/` | 1 | the omen card icon |

The complete set, so you can diff your replacements against it:

```text
Characters   companion husk iris liora merchant mira nadia oracle
             sable sovereign yafit yara
Sanctums     apothecary bed belltower chest door farm herbgarden kitchen
             lumberyard memorial obelisk quarry reflection reliquary sawmill
             shrine stoneworks storage sunbath teahouse temple wheatfield
World        altar ground rock statue tree
Expedition   candle floor_reflection floor_stones projectile skull
Wardens      mire_body stag_antlers stag_body sultan_body
UI           omen
```

!!! note "One creature sprite, twelve archetypes"

    All twelve `Creature.Archetype` variants share `Characters/husk.png` and are
    distinguished by a per-archetype tint applied in `Dungeon.SpawnCreature`. To
    give an archetype its own art you must add a load call, not a file — see
    [Adding content](adding-content.md).

## The import contract

`Editor/ArtImporter.cs` is an `AssetPostprocessor`. It only touches textures whose
path contains `/Resources/Art/`, and it forces the following on every one of them:

| Setting | Value | Why it matters |
| --- | --- | --- |
| Texture type | `Sprite` | a `Texture2D` reference will not resolve |
| Sprite mode | `Single` | no sheets, no sub-sprites |
| Pixels per unit | `400` under `/Characters/`, else `100` | sets the sprite's world size |
| Filter mode | `Bilinear` | smooth scaling, no pixel-art crunch |
| Mip maps | off | billboards are always near the camera |
| Alpha is transparency | on | required for the cut-out edges |
| Compression | `CompressedHQ` | keeps gradients clean |
| Mesh type | `FullRect` | tiled ground and scaled props need the full quad |
| Fallback physics shape | off | nothing uses sprite colliders |

Because the mesh is `FullRect`, `sprite.bounds` is the **whole rectangle**,
transparent pixels included. That has two direct consequences:

1. Keep the character's feet on the bottom edge of the canvas, or it will hover.
2. Do not add padding to make the canvas a convenient size — the padding counts.

## Step 1 — Replace the file, keep the name

1. Create your PNG at the exact path of the file you are replacing.
2. Keep the file name lowercase; the code builds it from an enum or a record name.
3. Delete the old PNG rather than adding `farm2.png` next to it — nothing loads `farm2`.
4. If the sprite is new (not a replacement), also wire it up; see
   [Adding content](adding-content.md).

```bash
# What is actually on disk, with pixel sizes
find Assets/ChosenOfTheHarvest/Resources/Art -name '*.png' \
  -exec file {} \; | sort
```

```powershell
# Same check on Windows
Get-ChildItem Assets\ChosenOfTheHarvest\Resources\Art -Recurse -Filter *.png |
  ForEach-Object {
    $img = [System.Drawing.Image]::FromFile($_.FullName)
    "{0}x{1}`t{2}" -f $img.Width, $img.Height, $_.FullName
    $img.Dispose()
  }
```

## Step 2 — Match the framing

The template's art was authored to one rhythm. Following it means no code changes.

| Slot | Shipped size | World result |
| --- | --- | --- |
| Characters | 1024 px tall, 338–665 wide | 2.56 units at 400 PPU; player is scaled to 0.75, followers to 0.6 |
| Sanctums | 768 px tall, 235–1248 wide | fitted to 1.6–2.8 units tall, clamped to 3 units wide |
| World props | 512–768 px | fitted to `targetHeight` (tree 3.2, rock 1.8, altar 2.6, statue 2.4) |
| Ground | 512 × 512 | 5.12 units at 100 PPU, tiled across the 40 × 30 hub |
| Omen icon | 343 × 512 | drawn in menus at text height |
| Wardens | 1024 px tall | fitted to 2.5–2.6 units, clamped to 2.8 units wide |

Characters are **not** fitted to a target height — the root scale is fixed, so PPU
is the only lever on their size. A 1024 px portrait at 400 PPU is 2.56 units; if you
export at 512 px and keep 400 PPU, the character is half height.

Buildings and props *are* fitted, so a different resolution is safe as long as the
aspect ratio is right. `Stage3D.CreateBillboard` uses
`min(targetHeight / height, maxWidth / width)`, so a wide building is limited by
width and ends up shorter than its neighbours.

## Step 3 — Force the reimport

Saving over a PNG usually triggers a reimport. When it does not (Unity still has the
old texture in memory), force it:

```csharp
// Assets/ChosenOfTheHarvest/Editor/ReimportArt.cs
using UnityEditor;
using UnityEngine;

public static class ReimportArt
{
    [MenuItem("Chosen of the Harvest/Reimport art")]
    public static void Run()
    {
        var guids = AssetDatabase.FindAssets(
            "t:Texture2D", new[] { "Assets/ChosenOfTheHarvest/Resources/Art" });

        foreach (var guid in guids)
        {
            string path = AssetDatabase.GUIDToAssetPath(guid);
            AssetDatabase.ImportAsset(path, ImportAssetOptions.ForceUpdate);
        }

        AssetDatabase.Refresh();
    }
}
```

The menu item lives under the template's existing top-level menu, next to the two
build entries in `BuildGame.cs`.

## Step 4 — Verify in play mode

1. Enter Play and look at the hub: player, followers, trees, rocks, altar, portal.
2. Press **B** and page through the build menu — every sanctum shows its icon.
3. Enter an expedition (**E** at the portal) and check floors, walls, doors, candles.
4. Reach a Warden and confirm the body sprite and its height.
5. Open the pause menu (**Esc**) and inspect the omen card icon in the deck.

!!! warning "Silent fallbacks hide mistakes"

    `SpriteForSanctum` falls back to `Sanctums/shrine` and unknown companions fall
    back to `Characters/companion`. A typo therefore looks like *the wrong sprite*,
    never like an error. If a building suddenly looks like a shrine, check the
    file name before anything else.

## Naming traps

**Enum → file name.** The sanctum path is the lowercased enum member:

```csharp
string art = type.ToString().ToLowerInvariant();
var sprite = Resources.Load<Sprite>("Art/Sanctums/" + art);
if (sprite == null) sprite = Resources.Load<Sprite>("Art/Sanctums/shrine");
```

So `TeaHouse` → `teahouse.png`, `WheatField` → `wheatfield.png`,
`BellTower` → `belltower.png`. Rename the enum to `TeaRoom` and the code looks for
`tearoom.png`; miss that and every tea house becomes a shrine.

**Record name → file name.** Companions load
`"Art/Characters/" + record.name.ToLowerInvariant()`. Names come from
`CompanionCatalog.Names`; to give a companion a portrait, add
`characters/<name>.png` in lowercase. `Amara` deliberately has no file and uses the
shared `companion.png`, so a missing portrait is not an error.

**Case on other platforms.** Windows resolves `Resources.Load` paths
case-insensitively; Linux and console builds do not. Always ship lowercase names
even if your local test passes with `Farm.png`.

**Warden parts.** `WardenAppearance.Init` loads only `stag_body`, `mire_body` and
`sultan_body`. `stag_antlers.png` ships with the template but is not referenced —
the Stag's antlers are part of its body sprite. Replacing the antlers file changes
nothing.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Building shows a shrine | file name does not match the lowercased enum member |
| Character floats or sinks | transparent padding above/below the feet in a `FullRect` sprite |
| Character too small or huge | PPU was not forced to 400 — the importer did not run |
| Sprite is invisible | texture imported as `Default`, not `Sprite` |
| Ground shows seams | tile is not square, or PPU is not 100 |
| Building looks squashed | aspect ratio changed while height fitting stayed on |
| Double shadow | a shadow is baked into the PNG; the code adds its own |
| Blurry at distance | mipmaps were enabled by a platform override |

## Related

- [Replacing audio](audio.md) — the same replace-in-place contract for sound.
- [Adding content](adding-content.md) — wiring a brand-new sprite into code.
- [Catalogs API](../reference/catalogs.md) — the data the art is attached to.
- [Troubleshooting](../troubleshooting.md) — import and rendering issues.
