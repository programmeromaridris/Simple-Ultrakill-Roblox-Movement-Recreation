# Roblox Movement System

An ULTRAKILL-inspired client-side movement framework for Roblox. All abilities are bound through a single central handler using `ContextActionService`, share a common `PlayerState` module for cross-ability awareness, and support both keyboard/mouse and gamepad inputs out of the box.

---

## Features

- **Sprint** — Variable walk speed toggled by holding shift
- **Dash** — Directional burst of speed with cooldown; falls back to look vector if the player is standing still
- **Slide** — Physics-based slide with steerable drift, hip height crouch, and configurable friction
- **Double jump & vault** — Two-jump budget with automatic wall-vault detection via raycast
- **Ground slam** — Midair downward slam with a spherical AOE hitbox
- **Parry** — Timed window that reflects enemy projectiles (integrates with the Enemy AI system)
- **Auto-fire** — `Heartbeat`-driven continuous fire for automatic weapons, cancelled cleanly on release or weapon switch
- **Centralized state** — `PlayerState` module prevents conflicting ability activations and is readable by both client and server modules

---

## Project Structure

```
ReplicatedStorage/
└── Modules/
    └── PlayerState          -- Shared state readable by client and server

StarterPlayerScripts/ (or equivalent LocalScript root)
└── MovementHandler          -- Central CAS binding script
└── MovementAbilities/
    ├── Sprint               -- Hold to run
    ├── Dash                 -- Directional burst dash
    ├── Slide                -- Physics slide with drift steering
    ├── Jump                 -- Double jump + vault override
    ├── GroundSlam           -- Midair AOE slam
    ├── Parry                -- Projectile reflect window
    └── M1                   -- Shoot / auto-fire handler
```

---

## Input Bindings

| Action | Keyboard | Gamepad |
|---|---|---|
| Sprint | Left Shift | L2 |
| Dash | Q | X |
| Jump | Space | *(default)* |
| Slide | C | *(unbound)* |
| Ground Slam | Left Control | *(unbound)* |
| Parry | Mouse Button 2 | *(unbound)* |
| Shoot / M1 | Mouse Button 1 | *(unbound)* |

All bindings are set in `MovementHandler` via `CAS:BindAction`. Mobile buttons are created automatically (`createTouchButton = true`).

---

## Ability Details

### Sprint

Increases `WalkSpeed` from 20 to 30 while held. Cancelled immediately if `PlayerState.CanAct()` returns false (e.g. during a stun or interaction).

| Config | Default |
|---|---|
| `NORMAL_SPEED` | 20 |
| `RUNNING_SPEED` | 30 |

---

### Dash

Fires a `BodyVelocity` in the player's current move direction (or look vector if standing still) for a short duration. Uses `LastDashTime` from `PlayerState` for cooldown tracking.

| Config | Default |
|---|---|
| `DASH_FORCE` | 100 |
| `DASH_DURATION` | 0.3s |
| `DASH_CD` | 1.5s |

---

### Slide

Triggered on press. Sets `CustomPhysicalProperties` on every character part to reduce friction, drops `HipHeight` to crouch, then launches a `BodyVelocity` forward. A `Heartbeat` loop lerps the velocity toward the player's move direction each frame for steerable drift. All original physical properties are restored after the slide ends.

`isSliding` is exposed in `PlayerState` so the weapon system can award the `"Sliding"` style bonus.

| Config | Default | Notes |
|---|---|---|
| `SLIDE_SPEED` | 80 | Initial and target velocity magnitude |
| `SLIDE_FRICTION` | 0.01 | 0 = ice, 1 = no slide |
| `SLIDE_DURATION` | 0.7s | How long the slide lasts |
| `SLIDE_CD` | 0.5s | Shares `LastDashTime` key |
| `TURN_SPEED` | 0.15 | Lerp alpha per frame — lower = wider drift |
| `DEFAULT_HIPHEIGHT` | 2 | Restored on slide end |

---

### Jump (Double Jump + Vault)

Overrides the default Roblox jump entirely. On the first press from the ground, manually applies vertical velocity and sets `Jumping` state. On subsequent presses, a short forward raycast (2.5 studs) checks for a wall:

- **Wall detected** → vault (lower upward boost, no jump count consumed beyond the normal budget)
- **No wall** → double jump (full boost)

Jump count resets to 0 when `HumanoidStateType.Landed` fires, wired in `MovementHandler`.

| Config | Default |
|---|---|
| `JUMP_POWER` | 70 |
| `DJUMP_POWER` | 70 |
| `VAULT_POWER` | 25 |
| `MAX_JUMPS` | 2 |

---

### Ground Slam

Only activates when the player is midair (`Jumping` state or `FloorMaterial == Air`). Fires a downward `BodyVelocity` and immediately checks a large spherical hitbox below the player. Any humanoids inside are reported to the server via `SlamHit:FireServer(hitHum)` for damage application.

| Config | Default |
|---|---|
| `SLAM_FORCE` | 100 |
| `SLAM_CD` | 1.5s |
| `HITBOX_SIZE` | 30 studs |

---

### Parry

On press, sets `isParrying` in `PlayerState`, fires `PlayerParrying:FireServer(true)`, and spawns an invisible `ForceField` on the character as a visual stand-in. Parry state is intentionally **not** gated by `CanAct()`, allowing players to parry while sliding. The server-side enemy projectile handler reads `parryingPlayer` to check whether a hit should be reflected.

| Config | Default |
|---|---|
| `PARRY_DURATION` | 5s |

---

### M1 (Shoot / Auto-fire)

Fires `ShootEvent:FireServer(mouse.Hit.Position)` on click. For `"Auto"` fire mode weapons, a `Heartbeat` loop fires continuously at the weapon's `FireRate` interval until mouse-up or weapon switch. Weapon stats are cached client-side via `WeaponEquipped.OnClientEvent` — no per-shot server round-trip for fire mode checks.

---

## PlayerState Reference

`PlayerState` is a shared module (stored in `ReplicatedStorage`) that holds the current flags for the local player. Both client ability modules and server-side systems read from it.

| State | Type | Set by |
|---|---|---|
| `isRunning` | bool | Sprint |
| `isDashing` | bool | Dash |
| `isSliding` | bool | Slide |
| `isParrying` | bool | Parry |
| `isSlamming` | bool | Ground Slam |
| `isInteracting` | bool | *(external — interaction system)* |
| `isStunned` | bool | *(external — combat system)* |
| `JumpCount` | number | Jump, MovementHandler |
| `LastDashTime` | number | Dash, Slide |
| `LastSlamTime` | number | Ground Slam |

`PlayerState.CanAct()` returns `true` when neither `isInteracting` nor `isStunned` is set. Most abilities gate on this before executing.

### Adding a New Ability

1. Create a new `ModuleScript` under `MovementAbilities` with an `.Execute(actionName, inputState, inputObject)` function.
2. Return `Enum.ContextActionResult.Sink` if the input was consumed, `Pass` to let it fall through.
3. Register it in `MovementHandler`:

```lua
local MyAbility = require(script.Parent.MovementAbilities.MyAbility)
CAS:BindAction("MyAbility", MyAbility.Execute, true, Enum.KeyCode.G)
```

4. Add any new state flags to `PlayerState.States` so other modules can react to them.

---

## Integration with Other Systems

| System | Integration point |
|---|---|
| **Weapon system** | `isSliding` in `PlayerState` → awards `"Sliding"` style bonus on hit |
| **Enemy AI** | `PlayerParrying` remote event → server reflects projectiles back |
| **Style manager** | `SlamHit` remote event → server applies slam damage and awards style |

---

## Known Limitations & TODOs

- `GroundSlam` reads `LastSlamTime` from `PlayerState` but never writes it back — the cooldown check always passes
- `Parry` does not check cooldown at all; a spammed right-click will reset the 5-second window repeatedly, used for debugging
- Slide cooldown shares `LastDashTime` with Dash, so dashing locks out sliding and vice versa — intentional or not, worth making explicit
- `Sprint` writes `isSprinting` to `PlayerState` but the initial state table defines it as `isRunning`; these should be unified
- Vault detection uses a single forward raycast at HRP height — low walls and sloped surfaces may not register reliably
- Ground slam hitbox fires the moment the slam input is pressed, not on landing — fast ceilings can trigger it before the player has descended

---

## License

MIT — use freely in your own Roblox projects.
