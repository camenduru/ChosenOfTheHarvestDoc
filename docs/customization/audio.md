# Replacing audio

Audio is plain `AudioSource` and `AudioClip` — no mixer, no middleware, no
ScriptableObject bank. `SoundDirector` owns two 2D sources and loads clips from
`Assets/ChosenOfTheHarvest/Resources/Audio/<key>.wav`, where `<key>` is one of 21
fixed names. Anything missing falls back to a synthesized tone, so the game never
goes silent while you work.

## The 21 keys

`SoundBank.UsedKeys` is the single source of truth. Files are loaded by exactly
these names, without the extension.

| Key | Kind | Fires from |
| --- | --- | --- |
| `music_hub` | loop | hub entry, returning from an expedition |
| `music_dungeon` | loop | every non-Warden expedition room |
| `music_boss0` | loop | Act I Warden room |
| `music_boss1` | loop | Act II Warden room |
| `music_boss2` | loop | Act III Warden room |
| `swing` | one-shot | melee attacks, bomb throw |
| `hit` | one-shot | damage to a creature, projectile impact |
| `hurt` | one-shot | damage to the player |
| `die` | one-shot | any death, including a Warden |
| `gather` | one-shot | harvesting a tree or rock |
| `build` | one-shot | placing or upgrading a sanctum, interior pieces |
| `pickup` | one-shot | collecting production, potions, chest loot |
| `chest` | one-shot | opening a reward chest |
| `ceremony` | one-shot | hymns, rites, repairs, omens, quest completion |
| `feast` | one-shot | feast and plenty ceremonies |
| `sacrifice` | one-shot | sacrifice ceremony, bomb detonation |
| `portal` | one-shot | entering or leaving an expedition, Warden teleports |
| `sharpen` | one-shot | sharpening at the forge |
| `ui` | one-shot | opening or choosing a menu entry |
| `error` | one-shot | refused actions and unaffordable builds |
| `slam` | one-shot | heavy Warden attacks |

```bash
# Every key that currently has a file
ls Assets/ChosenOfTheHarvest/Resources/Audio/*.wav |
  xargs -n1 basename | sed 's/\.wav$//' | sort
```

!!! note "Placeholder art is deliberate"

    The shipped WAVs are original synthesized tones, not extracted audio. They are
    safe to ship while prototyping and safe to delete when you replace them — the
    code regenerates an equivalent tone for any key whose file is absent.

## Step 1 — Replace the clip

1. Export your clip as `<key>.wav` using one of the 21 names above.
2. Drop it into `Assets/ChosenOfTheHarvest/Resources/Audio/`, overwriting the old file.
3. Do not rename the file to a friendly title; the key *is* the lookup path.
4. For a brand-new sound, add the key to `SoundBank.UsedKeys` first (see below).

```text
Resources/Audio/
  music_hub.wav        music_dungeon.wav
  music_boss0.wav      music_boss1.wav      music_boss2.wav
  swing.wav  hit.wav   hurt.wav   die.wav
  gather.wav build.wav pickup.wav chest.wav
  ceremony.wav feast.wav sacrifice.wav portal.wav
  sharpen.wav ui.wav   error.wav  slam.wav
```

## Step 2 — Match the format

The generated clips are mono at 22 050 Hz. Your files do not have to match, because
nothing resamples at runtime, but the shipped import settings are a useful baseline:

| Import setting | Shipped value | Effect |
| --- | --- | --- |
| Load type | `DecompressOnLoad` | decoded up front, no first-play hitch |
| Compression | Vorbis, quality 1 | small build, fine for short SFX |
| Preload audio data | off | saves a little memory at startup |
| Force to mono | off | stereo is preserved |
| Normalize | on | consistent perceived loudness |
| 3D | on in the meta, ignored at runtime | both sources use `spatialBlend = 0` |

Music clips are 24 s (`music_hub`, `music_dungeon`) and 16 s (the three Warden
themes). The synthesized versions fade the first and last 0.12 s so the loop point
is clickless; if you author a replacement, match its ends or accept a seam. The loop
itself is forced at runtime (`musicSource.loop = true`), so the importer's loop flag
does nothing.

!!! warning "One source means shared pitch"

    SFX play through a single `AudioSource` with `PlayOneShot`. Volume per call is
    respected, but pitch is a property of the *source*: setting a pitch for a new
    sound also retunes every one-shot still playing. Values are clamped to
    `0.55 – 1.9`, so extreme pitch tricks are not available without a second source.

## Step 3 — Verify what actually loaded

`SoundDirector` tracks four counters: `ClipCount`, `PlaceholderClipCount`,
`UsedClipCount` and `MissingPlaceholderCount`. The smoke test prints them:

```text
audio=True clips=21 placeholderAudio=21 usedAudio=21 missingAudio=0 archive=21
```

A quick manual check from the console or a test hook:

```csharp
var audio = HarvestGame.Instance.Audio;
Debug.Log($"clips={audio.ClipCount} used={audio.UsedClipCount} " +
          $"missing={audio.MissingPlaceholderCount} music={audio.CurrentMusic}");
```

`clips` counts everything in the dictionary, including synthesized fallbacks;
`placeholderAudio` counts only files found in `Resources/Audio`. If
`placeholderAudio` is lower than 21, a file name does not match its key.

## Step 4 — Tune the mix

There is no mixer asset. Volume is arithmetic in `SoundDirector.ApplyVolumes()`:

| Stage | Default | Where |
| --- | --- | --- |
| Master | `0.85` | `Preferences.MasterVolume` |
| Music | `0.65` | `Preferences.MusicVolume` |
| SFX | `0.85` | `Preferences.SfxVolume` |
| Music source | `0.34 × master × music` | `SoundDirector` |
| SFX source | `0.75 × master × sfx` | `SoundDirector` |

At the defaults that is roughly `0.19` for music and `0.54` for SFX. Per-call volume
multiplies on top and is clamped to `0 – 1`.

```csharp
// Change the balance for every source, then re-apply.
Preferences.SetMaster(0.8f);
Preferences.SetMusic(0.5f);
Preferences.SetSfx(0.9f);
Preferences.Apply();          // pushes the values into SoundDirector
```

Mute is separate from volume: `M` toggles `AudioSource.mute` on both sources and
persists to `PlayerPrefs` under `choth_audio_muted`. Music keeps playing silently
while muted, so unmuting resumes mid-track rather than restarting it.

## Adding a new sound

1. Add the key to `UsedKeys` (and a description to `Palette`, which documents the
   intended timbre and is otherwise unused).
2. Drop `<key>.wav` into `Resources/Audio/`.
3. Call `SoundDirector.Sfx("<key>", volume, pitch)` or
   `HarvestGame.Instance.Audio.Play("<key>", volume, pitch)`.

```csharp
// AudioArchive.cs
public static readonly string[] UsedKeys =
{
    "music_hub", "music_dungeon", "music_boss0", "music_boss1", "music_boss2",
    "swing", "hit", "hurt", "die", "gather", "build", "pickup", "chest",
    "ceremony", "feast", "sacrifice", "portal", "sharpen", "ui", "error",
    "slam", "forge"
};

public static readonly string[] Palette =
{
    // ... one entry per key, in the same order ...
    "heavy stone impact", "anvil ring"
};
```

```csharp
// Anywhere with a UnityEngine reference
SoundDirector.Sfx("forge", 0.75f, 1.1f);
HarvestGame.Instance.Audio.Play("forge", 0.6f, 0.9f);
```

```text
Resources/Audio/forge.wav
```

!!! warning "A key without a file is silent, not synthesized"

    The synthesized fallback exists only for the 21 built-in keys, because
    `BuildClips()` allocates them by name. A key added to `UsedKeys` with no
    matching WAV is counted in `MissingPlaceholderCount` and produces no sound at
    all. Ship the file with the key.

Music can be switched the same way. `PlayMusic` is a no-op if the requested key is
already playing, so it is safe to call every time a room loads:

```csharp
SoundDirector.Instance?.PlayMusic("music_dungeon");
```

## Related

- [Replacing art](art.md) — the same replace-in-place workflow for sprites.
- [Audio system](../guide/audio.md) — the design-level overview.
- [Balancing & tuning](balancing.md) — where the volume defaults sit.
- [Troubleshooting](../troubleshooting.md) — silent clips and import problems.
