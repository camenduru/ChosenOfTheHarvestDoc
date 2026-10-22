# License & credits

## Licence

**Chosen of the Harvest** is distributed under the
[Unity Asset Store EULA](https://unity.com/legal/as-terms). The short version:

| You can | You cannot |
|---|---|
| Use the template in commercial and non-commercial projects | Resell or redistribute the template itself, in whole or in part |
| Modify the code, art and audio | Publish the template as a template or asset pack |
| Ship a game built with it, royalty-free | Claim authorship of the original template |
| Use it in multiple of your own projects | Share it with people who did not buy it |

The licence is per-seat, as with all Asset Store products. A team needs a seat
per person using the package, per the EULA.

## Third-party components

The package ships **no third-party art, audio, fonts or code libraries**. The
only third-party components are Unity packages, declared in
`Packages/manifest.json` and governed by the Unity Companion License:

- Universal Render Pipeline
- Input System
- uGUI
- 2D Sprite
- Test Framework
- Rider / Visual Studio integrations

The HUD uses Unity's built-in `LegacyRuntime.ttf` resource. No font file is
redistributed.

The full list, with licence references, is in
`Assets/ChosenOfTheHarvest/Third-Party Notices.txt`.

## AI disclosure

**Artwork.** The 49 sprites under `Resources/Art` were generated with an
**open-source text-to-image diffusion model** from prompts written for this
project. They were then reviewed, background-removed, cropped, resized and
colour-corrected. No third-party image was used as an input or style reference.

**Audio.** Not AI-generated. The 21 WAV clips are produced by the project's own
synthesis code (`Scripts/Core/AudioManager.cs`).

**Code, text, UI, design.** Written for this project.

If you publish a game built on this template, describe the art honestly in your
own store listing. Do not call these images "hand-drawn" or "painted".

## Credits

| Role | Credit |
|---|---|
| Design, code, systems | Chosen of the Harvest project |
| Placeholder art | Generated with an open-source text-to-image model |
| Placeholder audio | Procedurally synthesised by the project |
| Engine and packages | Unity Technologies |

## Attribution

Attribution is **not required** by the Asset Store EULA. If you want to credit
the template anyway:

> Built with the *Chosen of the Harvest* Unity template.

## Reporting a licence problem

If you believe a file in this package infringes your rights, open an issue with
the file path and the basis of the claim:

<https://github.com/camenduru/ChosenOfTheHarvestDoc/issues>

## Related

- [Asset Store submission](publishing/asset-store.md)
- [Changelog](changelog.md)
- [FAQ](faq.md)
