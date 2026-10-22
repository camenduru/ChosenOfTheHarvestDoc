# Input & rebinding

Input runs on the Unity **Input System**. The template uses two layers: a small
rebindable action table and a set of fixed keys read directly from
`Keyboard.current`.

## The rebindable layer

`Scripts/Core/Controls.cs` defines the actions and their defaults:

```csharp
public enum Action
{
    Attack, Dodge, Interact, Build, Companions, Inventory, Map, Quest,
    Mute, Save, Load, Potion, Rite, Bomb
}

static readonly Key[] Defaults =
{
    Key.Space, Key.LeftShift, Key.E, Key.B, Key.F, Key.I, Key.C, Key.J,
    Key.M, Key.F5, Key.F9, Key.H, Key.X, Key.G
};
```

| Action | Default |
|---|---|
| Attack | `Space` |
| Dodge | `Left Shift` |
| Interact | `E` |
| Build | `B` |
| Companions | `F` |
| Inventory | `I` |
| Map | `C` |
| Quest | `J` |
| Mute | `M` |
| Save | `F5` |
| Load | `F9` |
| Potion | `H` |
| Rite | `X` |
| Bomb | `G` |

Bindings persist in `PlayerPrefs` under `choth_bind_<index>`. Reading is
lazy — `Load()` runs on first access.

```csharp
if (Bindings.Pressed(Bindings.Action.Attack)) player.Attack();
if (Bindings.Held(Bindings.Action.Dodge))    /* ... */;
```

`Pressed` returns true on the frame the key goes down; `Held` returns true while
it is held. Both are safe when no keyboard is connected.

### Add a rebindable action

1. Add the member to the `Action` enum.
2. Add the default `Key` at the **same index** in `Defaults`.

That is all — the persistence loop iterates over `Defaults.Length`, so the new
action is stored and rebindable automatically.

### Change or reset bindings

```csharp
Bindings.SetKey(Bindings.Action.Attack, Key.Q);
Bindings.ResetDefaults();
```

The **Preferences → Controls** screen calls these; it is built in `Hud.cs`.

## The fixed layer

These are read directly and are not rebindable, because they are tied to a
specific screen or mode:

| Input | Action | Where |
|---|---|---|
| `W` `A` `S` `D` / arrows | Move | `PlayerController.Update` |
| `Q` / `Z` | Cycle armament / rite | `HarvestGame.Update` |
| `K` | Catalog codex (hub only) | `HarvestGame.Update` |
| `R` | Manage nearby sanctum (hub only) | `HarvestGame.Update` |
| `F12` | Photo mode | `HarvestGame.Update` |
| `Esc` | Pause / close | `HarvestGame.Update`, `HarvestHud.Update` |
| `1`–`6` | Menu choice | `HarvestHud.Update` |

To make one of these rebindable, add it to `Bindings.Action` and replace the
direct `kb.someKey.wasPressedThisFrame` check with
`Bindings.Pressed(Bindings.Action.YourAction)`.

## Gamepad

`PlayerController` and `HarvestGame` read `Gamepad.current` when a controller is
connected:

| Input | Action |
|---|---|
| Left stick / D-pad | Move |
| South | Attack |
| East | Dodge |
| West | Interact |
| North | Build |
| Right shoulder | Rite |
| Left shoulder | Bomb |
| Start | Pause |
| D-pad up/down, South/East | Menus |

Gamepad bindings are fixed in code. The cleanest way to make them configurable is
to extend `Bindings` with an input-source abstraction rather than duplicating the
checks.

## Pointer input

The UI is uGUI and uses `EventSystem` raycasts for mouse clicks on menus. The
world itself is keyboard/gamepad only; there is no click-to-move. Placement mode
uses the cursor position projected onto the ground plane
(`HarvestGame.HandlePlacementInput`).

## Related

- [Controls](../getting-started/controls.md)
- [HUD reference](../guide/hud.md)
- [Preferences API](../reference/core.md)
