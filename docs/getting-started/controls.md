# Controls

Every control in the template, with the exact binding and where it is read.

## Keyboard

### Movement and combat

| Input | Action | Read in |
|---|---|---|
| `W` `A` `S` `D` / arrow keys | Move | `PlayerController.Update` |
| `Space` | Attack | `Bindings.Action.Attack` |
| `Left Shift` | Dodge (i-frames) | `Bindings.Action.Dodge` |
| `Q` | Cycle armament | `HarvestGame.Update` (fixed) |
| `Z` | Cycle rite | `HarvestGame.Update` (fixed) |
| `X` | Cast rite | `Bindings.Action.Rite` |
| `G` | Throw bomb | `Bindings.Action.Bomb` |
| `H` | Drink potion | `Bindings.Action.Potion` |

### Interaction and screens

| Input | Action | Read in |
|---|---|---|
| `E` | Interact | `Bindings.Action.Interact` |
| `B` | Build menu / interior build | `Bindings.Action.Build` |
| `R` | Manage nearby sanctum (hub) | `HarvestGame.Update` (fixed) |
| `F` | Companion roster | `Bindings.Action.Companions` |
| `I` | Inventory | `Bindings.Action.Inventory` |
| `K` | Catalog codex (hub) | `HarvestGame.Update` (fixed) |
| `C` | Pilgrimage map | `Bindings.Action.Map` |
| `J` | Quest log | `Bindings.Action.Quest` |
| `1`–`6` | Choose a menu option | `HarvestHud.Update` |
| `Esc` | Pause / close menu | `HarvestGame.Update`, `HarvestHud.Update` |

### System

| Input | Action | Read in |
|---|---|---|
| `F5` | Save | `Bindings.Action.Save` |
| `F9` | Load | `Bindings.Action.Load` |
| `F12` | Photo mode | `HarvestGame.Update` (fixed) |
| `M` | Mute | `Bindings.Action.Mute` |

## Gamepad

| Input | Action |
|---|---|
| Left stick / D-pad | Move and menu navigation |
| South (A / Cross) | Attack; confirm in menus |
| East (B / Circle) | Dodge; close menu |
| West (X / Square) | Interact |
| North (Y / Triangle) | Build |
| Right shoulder | Cast rite |
| Left shoulder | Throw bomb |
| Start | Pause |

Gamepad input is read through `UnityEngine.InputSystem.Gamepad.current`, so it
works with any controller the Input System supports. There is no gamepad
rebinding UI; bindings are fixed.

## Rebinding

Bindings for the actions in `Bindings.Action` are stored in `PlayerPrefs` under
the prefix `choth_bind_` and can be changed in **Preferences → Controls**.

```csharp
// Read the current binding
Key key = Bindings.GetKey(Bindings.Action.Attack);

// Rebind
Bindings.SetKey(Bindings.Action.Attack, Key.Q);

// Restore every default
Bindings.ResetDefaults();
```

`Bindings.Pressed(action)` returns true on the frame the key goes down;
`Bindings.Held(action)` returns true while it is held. Both no-op safely when no
keyboard is present.

!!! note "Fixed vs rebindable"

    Movement, `Q`, `Z`, `K`, `R`, `F12`, the menu digits and `Esc` are deliberately
    fixed because they are tied to screen layout or to a mode. Everything else is
    rebindable.

## Photo mode

`F12` toggles photo mode. While active:

- the HUD is hidden;
- `WASD` / arrows pan the camera;
- `Q` / `E` change the distance;
- `F12` captures a PNG next to the player log and returns to play.

## Related

- [Input & rebinding](../customization/input.md)
- [HUD reference](../guide/hud.md)
- [Your first run](first-run.md)
