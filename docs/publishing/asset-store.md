# Asset Store submission

This page is the packaging and listing checklist for publishing the template in
the Unity Asset Store **Templates** category, based on the
[Asset Store Submission Guidelines](https://assetstore.unity.com/publishing/submission-guidelines)
(last updated 20 May 2026) and the
[Asset Store Content Transparency](https://unity.com/legal/asset-store-content-transparency)
policy.

!!! info "Not legal advice"

    Requirements change. Re-read the guidelines before every submission.

## What the guidelines require of a Template

Templates are a distinct category with extra rules (§2.6):

| Rule | Requirement | How this template satisfies it |
|---|---|---|
| 2.6.a | Designed as an instructional, tutorial or framework product | The package is a framework plus a playable demo |
| 2.6.b | Visual content or functionality demonstrated in a demo scene | `Scenes/ChosenOfTheHarvest.unity` is the demo |
| 2.6.c | Documentation explains how the project is designed and how to extend it | Offline manual in `Documentation/` plus the online site |
| 1.1.f | Template submissions include demo scenes | One scene, fully playable |
| 1.3.c | Templates on Unity 6.5+ must support URP or HDRP | URP 17.6 |

## Package layout

One root folder, as §2.1.a requires:

```
Assets/
└── ChosenOfTheHarvest/
    ├── Documentation/           Markdown manual (ships)
    ├── Editor/
    ├── Resources/
    ├── Scenes/
    ├── Scripts/
    ├── Settings/
    └── Third-Party Notices.txt
```

`Resources` is nested under the root. `Resources.Load` finds any folder named
`Resources`, so a single root folder does not break the load paths.

## Pre-submission checklist

### Content

- [ ] The demo scene plays from a clean import with **zero console errors and
      warnings** originating from package content (§1.1.b).
- [ ] No duplicate, unused or redundant files (§2.1.c).
- [ ] No `AssetStoreTools` folder (§2.1.d).
- [ ] All asset paths are under 150 characters (§2.1.e).
- [ ] No executables, DLLs, `.apk`, `.app` or archives that hide content
      (§1.5.a, §2.2.a).
- [ ] No watermarks, DRM, time limits or feature locks (§1.4.b).
- [ ] The package does not phone home, proxy traffic or collect analytics
      (§1.1.g, §1.5.e).

### Code

- [ ] Every script is inside the `ChosenOfTheHarvest` namespace (§2.5.a).
- [ ] Editor menus are under **Tools → Chosen of the Harvest** (§2.5.1.a).
- [ ] No reflection into Unity Editor internal APIs (§2.5.g).
- [ ] No obsolete or deprecated API warnings on the target Unity version
      (§2.5.i).
- [ ] **Fast Enter Playmode** works with Domain Reload disabled (§2.5.h —
      mandatory from Unity 6.6). Every static field is reset in a
      `[RuntimeInitializeOnLoadMethod(SubsystemRegistration)]` method.
- [ ] The code is readable and intended to be modified (§2.5.d).

### Art and audio

- [ ] Sprites import as **Sprite (Single)**, sliced and named (§2.4.2.a).
- [ ] Images are lossless `.png` (§2.4.3.a).
- [ ] Template-category audio is lossless `.wav` (§2.7.c).
- [ ] SFX have no dead air or excessive noise (§2.7.d).

### Legal and disclosure

- [ ] `Third-Party Notices.txt` lists every third-party component and licence
      (§1.2.a).
- [ ] No GPL, LGPL or attribution-requiring CC/Apache dependency (§1.2.b).
- [ ] AI-generated content is disclosed in the listing's **AI description**
      field, naming the tool and the generated parts (§1.6.a).
- [ ] AI-generated art is not described as "drawn", "painted" or "hand-made"
      (§1.6.c).
- [ ] Marketing images are rendered in the Unity Editor; AI images are not the
      whole key image (§2.7.2, §2.7.6).
- [ ] Screenshots are included (§2.7.5).
- [ ] All listing metadata is in English (§2.4.1).

### Packaging

- [ ] `Library/`, `Logs/`, `Build/`, `BuildLogs/`, `UserSettings/` removed.
- [ ] `*.csproj`, `*.sln`, `*.log`, `.vs/` removed.
- [ ] The project opens in a clean Unity install without re-import prompts.
- [ ] The `.git` folder is not included.

## Cleaning a submission copy

Close Unity first, then:

```powershell
$root = "C:\path\to\ChosenOfTheHarvest"
Remove-Item -Recurse -Force "$root\Library","$root\Logs","$root\Build","$root\BuildLogs","$root\UserSettings","$root\obj","$root\.vs","$root\.git" -ErrorAction SilentlyContinue
Remove-Item -Force "$root\*.csproj","$root\*.sln","$root\import.log" -ErrorAction SilentlyContinue
```

Re-open the project once to confirm it imports cleanly, then upload.

## Listing metadata

| Field | Suggested value |
|---|---|
| **Title** | Chosen of the Harvest — Action Roguelike Template |
| **Category** | Templates → Systems / 2D |
| **Unity version** | 6000.6.6f1 |
| **Render pipeline** | Universal Render Pipeline |
| **Dependencies** | Input System, uGUI, URP (declared in the manifest) |
| **Tags** | roguelike, sanctuary, management, followers, procedural, save system, URP, 2D |

The description should state, near the top:

- that this is a **template/framework**, not a finished game;
- the Unity version and URP requirement;
- that the included art and audio are **replaceable placeholders**;
- the AI disclosure;
- a link to this documentation site.

!!! warning "Do not ship your own EULA"

    Template code is distributed under the **Unity Asset Store EULA**. Do not add
    a competing licence file for the code. You may add an AI disclosure and a
    third-party notices file, which this package does.

## Screenshots

Capture them from the running build, not from art tools. The bundled smoke test
does this automatically into `Documentation/Screenshots/`:

| Shot | Shows |
|---|---|
| `title.png` | Title screen |
| `hub.png` | The Harvesthold with companions and buildings |
| `companions.png` | The roster screen |
| `campaign.png` | The pilgrimage map |
| `dungeon.png` | An expedition room in combat |
| `boss.png` | A Warden fight |

Suggested set: one key image, six screenshots, one 30–60 second gameplay video
hosted externally (videos are not packaged — §2.3.c).

## Versioning policy

- Bump **major** for save-schema breaks or Unity-version jumps.
- Bump **minor** for new systems or content.
- Bump **patch** for fixes.
- Keep `CHANGELOG.md` at the project root and mirror it on the docs site.
- State the minimum Unity version in every release note.

## Support

The listing must give buyers a working support channel. This template points to
the documentation repository's issue tracker. Respond within a few business days
and keep a public changelog; the Asset Store review team checks responsiveness.

## Related

- [Building players](building.md)
- [License & credits](../license.md)
- [Changelog](../changelog.md)
