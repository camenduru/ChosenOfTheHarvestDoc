# Audio

`SoundDirector` is a single `MonoBehaviour` with two `AudioSource`s and no
middleware: one looping source for music, one `PlayOneShot` source for effects.
It synthesises a full placeholder palette in code at startup, then replaces each
clip with the same-named file from `Resources/Audio` when one exists. That means
the game is audible with zero audio assets, and shipping final audio is a file
swap rather than a code change.

## The 21 keys

`SoundBank.UsedKeys` is the authoritative list. It holds five music keys and
sixteen effect keys, in the order below.

| Key | Category | Shipped `.wav` | Generated fallback | Used for |
|---|---|---|---|---|
| `music_hub` | music | 8 s | 24 s | the Harvesthold, from `Awake` |
| `music_dungeon` | music | 8 s | 24 s | expedition rooms that are not Warden rooms |
| `music_boss0` | music | 8 s | 16 s | Ashen Orchard Warden |
| `music_boss1` | music | 8 s | 16 s | Drowned Conservatory Warden |
| `music_boss2` | music | 8 s | 16 s | Ember Citadel Warden |
| `swing` | SFX | 0.18 s | 0.22 s | every player attack |
| `hit` | SFX | 0.16 s | 0.20 s | a husk or projectile impact |
| `hurt` | SFX | 0.40 s | 0.42 s | the Sovereign takes damage |
| `die` | SFX | 0.80 s | 0.95 s | a husk or a Warden falls |
| `gather` | SFX | 0.30 s | 0.28 s | a resource node is struck |
| `build` | SFX | 0.50 s | 0.62 s | a sanctum is placed |
| `pickup` | SFX | 0.25 s | 0.36 s | loot, pearls, equipment |
| `chest` | SFX | 1.60 s | 0.95 s | a chest opens |
| `ceremony` | SFX | 2.40 s | 2.80 s | ceremonies, oaths, Warden defeats |
| `feast` | SFX | 1.40 s | 2.40 s | Feast and Ceremony of Plenty |
| `sacrifice` | SFX | 1.20 s | 1.90 s | a sacrifice is accepted |
| `portal` | SFX | 0.80 s | 1.25 s | entering or leaving an expedition |
| `sharpen` | SFX | 0.35 s | 0.55 s | Sharpen blade at the altar |
| `ui` | SFX | 0.08 s | 0.09 s | menu ticks and confirmations |
| `error` | SFX | 0.20 s | 0.28 s | a refused action |
| `slam` | SFX | 0.45 s | 0.65 s | Bell Golem and Warden ground slams |

The "generated fallback" column is the duration passed to `Make(...)`; the
shipped durations are what the mono 22,050 Hz files in `Resources/Audio`
measure. The shipped file always wins, so those are the lengths you hear in a
clean checkout.

!!! note "The palette is prose, not data"

    `SoundBank.Palette` is a parallel array of one-line descriptions ("warm
    harp and low strings", "mire queen reeds"). `SourceOf(key)` returns the
    description at the same index, and nothing in the game branches on it. It
    exists so the audio intent survives in source control while the clips are
    placeholders.

## Where clips come from

Two passes fill the same dictionary in `Awake`:

```csharp
void Awake()
{
    Instance = this;

    musicSource = gameObject.AddComponent<AudioSource>();
    musicSource.playOnAwake = false;
    musicSource.loop = true;
    musicSource.spatialBlend = 0f;
    musicSource.volume = 0.34f;

    sfxSource = gameObject.AddComponent<AudioSource>();
    sfxSource.playOnAwake = false;
    sfxSource.spatialBlend = 0f;
    sfxSource.volume = 0.75f;

    BuildClips();

    Muted = PlayerPrefs.GetInt(MutePref, 0) == 1;
    ApplyMute();
    ApplyVolumes();
}
```

`BuildClips()` synthesises all 21 clips with `AudioClip.Create(...)` at a fixed
`Rate = 22050`, mono, using sine/square/saw/noise generators. `Noise(int)` is a
deterministic hash, so every launch sounds identical. Music gets a 0.12 s fade
on both edges (`FadeEdges`) so an 8-second loop does not click.

Then `LoadPlaceholderClips()` overwrites them from Resources:

```csharp
foreach (var key in keys)
{
    var clip = Resources.Load<AudioClip>("Audio/" + key);
    if (clip == null)
    {
        MissingPlaceholderCount++;
        continue;
    }
    clips[key] = clip;
    PlaceholderClipCount++;
}
```

The naming convention is therefore exact and extension-free: a file at
`Assets/ChosenOfTheHarvest/Resources/Audio/music_hub.wav` loads as
`Resources.Load<AudioClip>("Audio/music_hub")`. A renamed file simply does not
load, and the generated clip keeps playing.

Four counters expose the result, which the smoke test logs:

| Property | Meaning |
|---|---|
| `ClipCount` | keys currently in the dictionary — 21 in a clean project |
| `UsedClipCount` | `SoundBank.UsedKeys.Length` — always 21 |
| `PlaceholderClipCount` | keys whose clip came from `Resources/Audio` |
| `MissingPlaceholderCount` | keys that fell back to the synth |

## Music vs SFX

The two sources have different jobs, and only the music source is asked to
loop.

| Source | Loop | Volume base | API | Clips |
|---|---|---|---|---|
| `musicSource` | yes | 0.34 | `PlayMusic(key)` | the five `music_*` keys |
| `sfxSource` | no | 0.75 | `Play(key, volume, pitch)` / `Sfx(...)` | the sixteen effect keys |

`PlayMusic` is idempotent: it returns early when the requested key is already
playing, so calling it every room does not restart the track.

```csharp
public void PlayMusic(string key)
{
    if (musicSource == null) return;
    if (!clips.TryGetValue(key, out var clip) || clip == null) return;
    if (currentMusic == key && musicSource.isPlaying) return;

    currentMusic = key;
    musicSource.clip = clip;
    musicSource.loop = true;
    musicSource.Play();
}
```

The call sites are few enough to list in full:

| Moment | Key | Where |
|---|---|---|
| Boot, before the title screen | `music_hub` | `HarvestGame.Awake()` |
| Enter an expedition | `music_dungeon` | `HarvestGame.EnterExpedition(int)` |
| Every room transition | `music_boss<act>` in a Warden room, else `music_dungeon` | `Expedition.EnterRoom` (in `Dungeon.cs`) |
| Warden dies | `music_dungeon` | `Expedition` boss-death routine |
| Return to the hub | `music_hub` | `HarvestGame.ExitExpedition(bool)` |

`music_boss0/1/2` are selected by act index with string concatenation —
`"music_boss" + ActIndex` — so a fourth act needs a fourth key and clip.

## Effects and the call pattern

Effects go through a static helper, which keeps combat code free of null checks:

```csharp
public static void Sfx(string name, float volume = 1f, float pitch = 1f)
{
    if (Instance != null) Instance.Play(name, volume, pitch);
}

public void Play(string name, float volume = 1f, float pitch = 1f)
{
    if (Muted || sfxSource == null) return;
    if (!clips.TryGetValue(name, out var clip) || clip == null) return;

    sfxSource.pitch = Mathf.Clamp(pitch, 0.55f, 1.9f);
    sfxSource.PlayOneShot(clip, Mathf.Clamp01(volume));
}
```

Three behaviours follow from that:

- **Pitch is a shared property, not per-shot.** `PlayOneShot` inherits the
  source pitch, so two overlapping effects use the pitch of the most recent
  call. Variation is still useful for single hits.
- **Unknown keys fail silently.** No log, no exception — a typo is a missing
  sound, not a crash.
- **Volume is clamped to 0-1** and multiplied by the source volume routing
  below, so passing `1f` does not mean "full scale".

Typical arguments in the source: `ui` at `0.55f`, a combo swing at `0.7f` with
pitch `0.85f + comboIndex * 0.04f`, `die` at `1f` and pitch `0.62f` for a Warden,
`error` at `0.6f`.

## Volume routing

Volumes are multiplied in two stages: a per-source base, then the player's
master and channel values.

```csharp
void ApplyVolumes()
{
    if (musicSource != null) musicSource.volume = 0.34f * masterVolume * musicVolume;
    if (sfxSource != null) sfxSource.volume = 0.75f * masterVolume * sfxVolume;
}
```

| Value | Default | Source |
|---|---|---|
| `masterVolume` | 0.85 | `Preferences.MasterVolume`, key `choth_settings_master` |
| `musicVolume` | 0.65 | `Preferences.MusicVolume`, key `choth_settings_music` |
| `sfxVolume` | 0.85 | `Preferences.SfxVolume`, key `choth_settings_sfx` |
| music base | 0.34 | `Awake` |
| SFX base | 0.75 | `Awake` |

With default settings that is an effective music level of
`0.34 × 0.85 × 0.65 ≈ 0.188` and SFX of `0.75 × 0.85 × 0.85 ≈ 0.542`. The SFX
source is intentionally the louder of the two.

`Preferences.Apply()` pushes the three values on boot and after every change:

```csharp
SoundDirector.Instance?.SetVolumes(MasterVolume, MusicVolume, SfxVolume);
```

The settings menu does not use a slider. `HarvestGame.ShowPreferences()` cycles
each volume in ten-point steps through `CycleVolume`, which rounds to a tenth,
adds one, wraps at 11 and divides back:

```csharp
static float CycleVolume(float value)
{
    int step = Mathf.RoundToInt(value * 10f);
    step = (step + 1) % 11;
    return step / 10f;
}
```

Pressing the row eleven times returns to where it started, and `0%` is reachable.

## Mute

++m++ is a binding, not a hard-coded key: `Bindings.Action.Mute` defaults to
`Key.M` and can be reassigned. `HarvestGame.Update` calls `ToggleMute()`, which
delegates to the director and toasts the new state:

```csharp
public void ToggleMute()
{
    bool muted = Audio.ToggleMute();
    HarvestHudRef.Toast(muted ? "Audio muted   •   press M" : "Audio on   •   press M");
}
```

Mute is stored separately from the settings block, in
`choth_audio_muted`, and survives restarts:

```csharp
const string MutePref = "choth_audio_muted";

public bool ToggleMute()
{
    Muted = !Muted;
    PlayerPrefs.SetInt(MutePref, Muted ? 1 : 0);
    PlayerPrefs.Save();
    ApplyMute();
    return Muted;
}
```

`ApplyMute()` sets `mute` on both sources rather than zeroing their volumes, so
un-muting restores the exact levels the player had set. `Play` also returns
early while muted, which keeps `PlayOneShot` calls from stacking up silently.

!!! warning "Mute is not a volume row"

    Mute is not part of `Preferences` and does not appear in the settings menu.
    Only ++m++ (or `SoundDirector.SetMuted`) changes it. If you add a "Mute"
    row to the menu, drive it through `SetMuted`, not `Preferences`, or the two
    states will disagree after a restart.

## Adding or replacing a clip

**Replacing** is a file swap. Keep the base name identical to the key, drop the
file into `Assets/ChosenOfTheHarvest/Resources/Audio/`, and let Unity import it
as an `AudioClip`. No code changes, no array edits, and the old synth version
stops being used automatically. Music files should be loop-safe; the source has
`loop = true`, and the code does not re-apply edge fades to imported clips.

**Adding** takes three steps:

1. Add the key to `SoundBank.UsedKeys`. This is what `LoadPlaceholderClips`
   iterates, so a key that is not in the list is never loaded from Resources.
2. Optionally add a matching description at the same index in
   `SoundBank.Palette`. `SourceOf` tolerates a short palette, but keeping the
   two arrays aligned is the point of the file.
3. Ship `Resources/Audio/<key>.wav` (or `.ogg`) and call it:

```csharp
SoundDirector.Sfx("bell", 0.8f, 1.1f);
```

If the file is missing, the key still exists in the dictionary only if
`BuildClips` made one — a brand new key with no synth clip and no file will be
silently skipped by `Play`. For a new key without a resource, either write a
`Make` entry in `BuildClips` or accept the silence until the asset lands.

!!! tip "Keep the key list stable"

    The 21 names are referenced by string across combat, the hub and the UI.
    Renaming a key means finding every `Sfx("…")` call; adding keys is cheap,
    renaming them is not.

## Related

- [Replacing audio](../customization/audio.md) — asset import settings and mixing
- [HUD and menus](hud.md) — where the volume rows and the mute toggle live
- [Localization](localization.md) — `en.json` strings for the settings labels
- [Bindings and defaults](../reference/core.md) - the default ++m++ mute binding
