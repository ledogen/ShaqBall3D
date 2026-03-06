# ShaqBall 3D — Documentation

---

## Glossary / Nicknames

These are informal names used during development that map to specific in-game concepts.

| Nickname | What it refers to |
|---|---|
| **the ball / ShaqBall** | The player-controlled sphere, textured with Shaq's face. Radius 0.5 units, mass 1.0 kg. |
| **the stage** | The entire tilting platform (`stageGroup`) the ball rolls on. 20 units wide (X: -10 to +10), base length 100 units (grows with difficulty). |
| **the ramp** *(Test Lab)* | Concrete ramp in the Test Lab, Z=15–35, rising from Y=0 to Y=5 (~14° slope). Near end buried below floor for a seamless bottom transition. Far end flush with the high platform. |
| **the high platform** *(Test Lab)* | Raised default-material platform at the far end of the ramp. Top surface at Y=5, 20 units long (Z=35–55), 14 units wide, with metal side rails. |
| **prop zone / prop spawn area** *(Test Lab)* | A 15×15 unit square at ground level (Y≈0.05) starting 3 units past the high platform (Z=58–73). Marked with metal corner posts. Props are spawned here via the Test Lab side panel. |
| **props** | Spawnable interactive obstacles placed in the prop zone via the Test Lab panel. Currently: Spinner, Launcher. |
| **warning zone / danger zone** | The translucent red fan-shaped mesh on the floor showing the arc the launcher arm will sweep through. Non-collidable, purely visual. |
| **the flip** | The forward swing phase of the launcher arm state machine (`swing` state). |
| **the toroid / half-pipe** | A `THREE.TorusGeometry` with arc = π (half circle), rendered `DoubleSide` so the inner curved surface is visible. Random XYZ rotation per spawn. |
| **spinner** | Rotating obstacle: metal hub cylinder + 3 tall concrete paddles at 120° intervals. Rotates around its Y axis each frame. |
| **launcher** | Swing-arm obstacle: metal base + concrete arm that sweeps ~153° when triggered. State machine: idle → windup → swing → return → cooldown. |
| **icyhot** | The goal collectible — a textured cylinder that floats above the stage, bobs and rotates, and moves toward the player when within attract radius. Collecting it advances the level. |

---

## Architecture Overview

### Scene Graph
- `scene` — root Three.js scene
  - `stageGroup` — the entire tilting stage, pivots around ball contact point each frame
    - Floor mesh (rebuilt each level)
    - All stage geometry (ramps, patches, obstacles, powerups)
    - Spinner groups
    - Launcher groups
    - Icyhot mesh (when active)
    - Powerup groups
  - `ballMesh` — Shaq-textured sphere, positioned in world space
  - `sun` — directional light tracking ball position for shadows
  - `gridHelper` — fixed world-space reference grid at Y=-10
  - `jetPoints` — particle system for jetpack exhaust

### Game Modes
- `'menu'` — orbiting camera, no physics, main menu overlay visible
- `'play'` — normal gameplay, random stage generated via `buildStage()`
- `'test'` — Test Lab layout via `buildTestLab()`, no powerup pickups, side panel visible

---

## Physics System

### Core Constants
| Parameter | Value |
|---|---|
| Ball mass | 1.0 kg |
| Ball radius (base) | 0.5 units |
| Gravity (base) | 25 m/s² |
| Max tilt | 15° |
| Tilt speed | 80 °/s |
| Tilt return speed | 120 °/s |
| Jump speed (base) | 9.6 m/s |
| Death threshold | Y < -15 |

### Collision
Per-triangle brute-force sphere vs mesh test (`sphereMeshCollision`). Every frame, `stageGroup.traverse` tests all meshes with `collidable !== false`. Finds deepest penetrating triangle, returns `{normal, depth}` via a reused `_hitResult` object (no heap allocation). Ball position is depenetrated: `pos += n * depth`. `onFloor = true` when `n.y > 0.643`.

### Bounce Response
Normal surfaces: `vel += n * -vn * (1 + restitution)`, clamped to prevent re-penetration. Bounce pads bypass restitution and apply preset `bounceVel = 25` upward when `n.y > 0.5`.

### Surface Velocity (Props)
When the ball contacts a spinner paddle or launcher arm, the world-space velocity of the contact point on the rotating body is computed as `ω × r_contact`. The impulse is applied in the relative velocity frame so the ball is genuinely launched rather than just deflected.

### Floor Velocity Compensation
Each frame, the stage matrix is snapped before and after the tilt quaternion is applied (`prevStageMat`, `currStageMat`). On each collision, the contact point is transformed through both matrices to compute the floor's world-space velocity, which is subtracted from relative velocity before impulse resolution. Suppressed for one frame after level transitions (`levelJustAdvanced`).

### Friction
Impulse-based. Slip velocity = (linear vel + ω×r) projected onto contact plane. Impulse capped at `μ * Fn * dt`. Applied to both `vel` and `angVel`. `I_cur = 0.4 * mass * r²` computed once per contact and shared between friction and rolling resistance.

### Rolling Resistance
Braking torque on `angVel` proportional to `Crr * Fn * r * dt / I`, capped at current `|ω|`.

### Jump Buffer
Space press buffered for 200ms — pressing space in the air triggers jump on next landing within the window. One jump per key press (rising edge detection, no OS key-repeat re-trigger).

---

## Performance Optimizations

All per-frame heap allocations in the physics hot path have been eliminated via module-level pre-allocated scratch objects:

- `closestPointOnTri` — 4 scratch vectors (`_ap`, `_bp`, `_cp`, `_CB`)
- `sphereMeshCollision` — reused `_hitResult` object, scratch vectors for closest point and normal
- Collision response — 19 scratch vectors covering all intermediate calculations
- `_stageMatInv` — stage matrix inverse computed once per frame before `traverse`, not per mesh
- `_gravVec` — gravity vector updated in `applyEffects`, not allocated per frame
- `prevStageMat` / `currStageMat` — module-level Matrix4s filled with `.copy()`, no `.clone()` per frame
- `_pivotTmp` — stage tilt pivot calculation, replaces `pivot.clone()`
- `_orientQ` — orientation quaternion update, replaces `new THREE.Quaternion()` per frame
- `_pickupWorld` — powerup world position check, replaces per-frame `new THREE.Vector3()`
- `_camTarget` — camera lerp target, replaces per-frame `new THREE.Vector3()`
- `_prevPos` — distance tracking, replaces `pos.clone()` per frame
- `_angVelAxis` — axis-angle normalization, replaces `angVel.clone()` per frame

`stageGroup.updateMatrixWorld(true)` is called exactly 3 times per frame: before tilt (for `prevStageMat`), after tilt (for `currStageMat`), and after spinner/launcher rotation (for collision, powerup pickup, and icyhot detection).

---

## Materials

| Material | Friction | Restitution | Rolling Resist | Notes |
|---|---|---|---|---|
| default | 0.6 | 0.55 | 0.1 | Brown checkerboard texture |
| wood | 0.6 | 0.3 | 0.016 | Plank texture with grain |
| ice | 0.05 | 0.1 | 0.002 | Very slippery, low bounce |
| rubber | 0.9 | 0.665 | 0.04 | High grip, moderate bounce |
| metal | 0.4 | 0.5 | 0.008 | Brushed metal look |
| concrete | 0.7 | 0.2 | 0.02 | Rough, low bounce |
| sand | 0.8 | 0.05 | 0.3 | Very high rolling resistance |
| wall | 0.5 | 0.3 | 0.01 | Transparent blue, stage side walls |
| bouncepad | 0.4 | 0.1 | 0.02 | Bypasses restitution; applies bounceVel=25 upward |

The last collided material name is displayed in the HUD (top-right) in its corresponding color via `MAT_HUD_COLOR`. Updates every frame during collision traversal.

---

## Powerups

Spawn as bobbing/rotating 3D models in play mode (2–3 per level). Not present in Test Lab as pickups — toggled live via the Test Lab side panel instead. Each powerup has a distinct 3D model rather than a generic sphere.

### Powerup Table

| Powerup | Color | Model | Effect |
|---|---|---|---|
| Size Up | Gold `#f0c040` | Sphere (fallback) | Ball radius ×1.5, jump speed ×1.5 |
| Bouncy Ball | Cyan `#40e0ff` | Sphere | Restitution multiplier ×1.6 (capped 0.99) |
| Speed Up | Orange-red `#ff6040` | 3 stacked chevron arrows | Gravity ×1.8 (faster roll), jump ×2.0 |
| Jetpack | Green `#80ff80` | Two pill capsules side by side | Space in air applies 18 m/s² thrust. Fuel drains at 0.45/s, refills at 0.4/s after 2s on ground. |
| Magnet | Pink `#ff44cc` | Horseshoe U-shape (half-torus + pole caps) | Doubles icyhot attract radius (10→20) and attract speed (2.3→4.6 u/s) |
| Low Gravity | Yellow `#ffff00` | Feather (shaft + vane + barbs) | Gravity ×0.5 (halved). Stacks multiplicatively with Speed Up. |

### Powerup Models (`buildPowerupMesh`)
Each type returns a `THREE.Group`. `castShadow` is set on all child meshes. Groups share the same bob (`position.y = 1.5 + sin(...)`) and Y-rotation (`+= 1.5 * dt`) animation applied to the group root each frame.

- **Bouncyball** — `SphereGeometry(0.6, 16, 16)`
- **Jetpack** — two sub-groups each containing a `CylinderGeometry` body + two `SphereGeometry` hemisphere caps, offset ±0.22 on X
- **Speed Up** — three `ExtrudeGeometry` chevron shapes from a `THREE.Shape`, stacked 0.3 apart on Z, rotated flat on XZ plane
- **Magnet** — `TorusGeometry(0.42, 0.18, 12, 24, Math.PI)` rotated 90° on X, plus two `CylinderGeometry` pole caps rotated 90° on X at ±0.42 X, offset −0.16 on Z
- **Low Gravity** — `BoxGeometry` shaft + scaled `SphereGeometry` vane + 10 small `BoxGeometry` barb boxes angled outward

---

## Icyhot Goal

A textured cylinder (`icyhot.png`) with emissive blue glow. Floats at Y=5 near the far end of the stage.

- **Attract radius** — 10 units (doubled to 20 with Magnet powerup)
- **Attract speed** — 2.3 u/s (doubled to 4.6 with Magnet powerup)
- **Collection radius** — 1.0 unit (world-space distance to ball)
- Moves in full 3D toward ball local position each frame when within attract radius
- Bobs ±0.15 units on Y using `sin(performance.now() * 0.001)`
- Slowly rotates on Y and Z axes each frame
- On collection: white flash, `difficulty++`, stage rebuild, ball reset, timer starts

### Radar Ring
A 2D canvas overlay (`radar-canvas`) draws a pulsing ring centered on the icyhot's projected screen position. The ring shrinks from `ICYHOT_RADAR_MAX * pxPerUnit` to 8px radius over 0.6 seconds, cycling continuously. Fades in as it shrinks. Hidden when icyhot is behind camera.

---

## Obstacles

### Spinners
Rotating `THREE.Group` — metal hub cylinder + 3 tall concrete paddles at 120° intervals. `group.rotation.y += speed * dt` each frame. Scale 0.8–1.4×, speed 0.8–2.5 rad/s × `(1 + difficulty * 0.15)`, random CW/CCW. 1–3 per play mode level.

### Launchers
Swing-arm obstacle with a sector-based trigger zone and state machine.

**Structure:**
- Metal base cylinder (scale 0.9–1.3×)
- Pivot group at Y=0.6×scale
- Concrete arm (4.5×scale long) extending from pivot, collision-active

**State Machine:**
| State | Duration | Behavior |
|---|---|---|
| idle | — | Waiting for ball to enter trigger sector |
| windup | `max(0, 0.8 - d * 0.06)`s | Arm flashes orange, no movement |
| swing | until arc complete | Arm sweeps 153° at 5.04 rad/s |
| return | until back at rest | Arm returns at 1.5 rad/s |
| cooldown | 1.0s | Locked, then back to idle |

Windup time reaches 0 at difficulty ~13 (instant fire, no warning).

**Trigger / Warning Zone:**
- Both use the same sector: `innerR = 0.3×scale`, `outerR = 5.0×scale`, arc = 153°
- Trigger converts ball to group-local space and checks radius + angular bounds
- Warning zone is a translucent red (`0xff2200`, opacity 0.22) fan mesh at floor level

### Half-Torus (Toroid / Half-pipe)
`THREE.TorusGeometry` with arc = π. `DoubleSide` material so inner surface renders. Randomised major radius (2–4), tube radius (0.5–1), full random XYZ rotation. 60% spawn probability per play mode level.

### Ramps
Thin tilted `BoxGeometry` slabs (wood/metal/concrete). Random width, length, tilt axis, and Z position.

### Bumpers
Tall cylinders (rubber/metal/concrete) acting as fixed posts. Random radius and height.

### Surface Patches
- **Sand patches** — high rolling resistance zones, up to 8 units wide
- **Ice patches** — very low friction zones

### Bounce Pads
Flat red cylinders on the floor. Contact when `n.y > 0.5` launches ball upward at 25 m/s regardless of approach velocity.

---

## Level Progression

Levels are generated by `buildStage()` using `difficulty` as the seed for all scaling parameters. `difficulty` increments each time the icyhot is collected.

| Parameter | Scaling |
|---|---|
| Stage length | `min(100 + d * 20, 600)` units |
| Obstacle count | `base * (0.85 + d * 0.25)` |
| Surface mix | d<3: default/wood/rubber; d<6: adds ice/sand; d≥6: ice/sand/rubber heavy |
| Powerup frequency | `max(1.0 - d * 0.08, 0.2)` |
| Spinner speed | base range × `(1 + d * 0.15)` |
| Launcher windup | `max(0, 0.8 - d * 0.06)` seconds |
| Timer | No timer on level 1. Level 2+: `30 + (d - 1) * 2` seconds |

Timer display turns red and pulses in the final 10 seconds.

---

## Run Stats

Tracked per run in play mode, displayed on the game over screen.

| Stat | How tracked |
|---|---|
| Max vertical | Peak `pos.y` during run |
| Longest airtime | Longest continuous stretch of `!onFloor`, accumulated via `dt` |
| Top speed | Peak `vel.length()` while `pos.y > -1` (excludes freefall) |
| Distance rolled | Cumulative `pos.distanceTo(prevPos)` per frame |
| Jumps | Incremented on each successful jump |
| Power-ups collected | Incremented on each pickup |

---

## Test Lab

Accessible via the ⚗ Test Lab button on the main menu. Fixed environment for physics and prop testing. No random obstacles. R resets the lab without generating a new random level.

### Layout (Z-axis, world space)
| Zone | Z range | Y | Description |
|---|---|---|---|
| Spawn zone | -10 to +15 | 0 | Bare stage floor, ball spawns at origin |
| Ramp | 15 to 35 | 0→5 | Concrete, 8 units wide, ~14° slope |
| High platform | 35 to 55 | 5 | Default material, 14 wide, metal side rails |
| Gap | 55 to 58 | — | Open air between platform and prop zone |
| Prop zone | 58 to 73 | ≈0 | 15×15 concrete square, corner posts |

### Test Lab Side Panel
Fixed right-side panel, hidden in play mode. Two sections:

**Powerups** — click to toggle each effect on/off instantly. Jetpack restores full fuel on enable. All effects reset when R is pressed.

| Toggle | fx key |
|---|---|
| Size Up | `sizeUp` |
| Bouncy Ball | `bouncyBall` |
| Speed Up | `speedUp` |
| Jetpack | `hasJetpack` |
| Magnet | `magnet` |
| Low Gravity | `lowGravity` |

**Props** — each click spawns one prop at a random position within the prop zone.
- `+ Spinner` — random scale (0.8–1.2×), random speed and direction
- `+ Launcher` — random yaw orientation
- `Clear props` — removes all spawned props and spinners/launchers instantly

---

## Camera

Spherical orbit around ball. Parameters:

| Constant | Value |
|---|---|
| `CAM_DIST` | 17.6 units |
| `CAM_AZ_SPEED` | 1.8 rad/s |
| `CAM_POL_DEFAULT` | 0.85 rad |
| `CAM_POL_HIGH` | 0.15 rad (arrow up) |
| `CAM_POL_LOW` | 1.35 rad (arrow down) |

Lerps to target position at 8% per frame. `snapCamera()` teleports instantly (used on reset and level transitions). Menu mode orbits automatically at 0.15 rad/s.

---

## Visual Systems

### Shadows
Ball-tracking directional shadow. Sun position and target follow ball each frame. 2048×2048 PCFSoft shadow map, ±30 unit frustum. Edge hidden inside fog.

### Fog
`THREE.Fog` matched to background `#333333`. Starts at 30 units, full at 70. Hides stage edges and shadow frustum boundary.

### Reference Grid
Fixed world-space `GridHelper` at Y=-10, 700 units wide. Does not tilt with stage.

### HUD (top-right)
- Level indicator (play mode only)
- Last collided material name, colored per `MAT_HUD_COLOR`
- Active powerup indicators with emoji and color
- Jetpack fuel bar (bottom-center, visible only when jetpack active)

### Timer Display
Fixed top-center. Hidden when no timer active. Turns red, pulses at opacity `0.7 + 0.3*sin(t)` in final 10 seconds.

### Collect Flash
Full-screen white overlay (`#collect-flash`) flashes on icyhot collection. Fades out over 0.6s via CSS transition.

### Jetpack Particles
Point cloud (`THREE.Points`) of up to 60 smoke particles emitted below ball while thrusting. Particles drift downward, fade after 0.3–0.55s. Material opacity fades when not thrusting.

### Shaq Dialog
RPG-style dialog box anchored to bottom of screen. Shows portrait image + character name + message text. Pauses the game loop (`dialogPaused = true`) until dismissed with any key or click. Used for intro and level 2 transition messages.

### Performance Diagnostics
TAB toggles overlay. Shows: frame time avg/max, FPS, collidable triangle count, draw calls, stage mesh count, powerup count, geometry/texture memory, JS heap. Color-coded green/yellow/red. Updates every 20 frames.

---

## Controls

| Key | Action |
|---|---|
| W / S | Tilt stage forward / backward (camera-relative) |
| A / D | Tilt stage left / right (camera-relative) |
| ← → | Orbit camera left / right |
| ↑ ↓ | Snap camera polar angle low / high |
| Space | Jump (200ms buffer) / Jetpack thrust |
| R | Reset ball and rebuild stage |
| TAB | Toggle performance diagnostics |
