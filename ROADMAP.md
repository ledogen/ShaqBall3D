# ShaqBall 3D — Documentation & Roadmap

---

## Glossary / Nicknames

These are informal names used during development that map to specific in-game concepts.

| Nickname | What it refers to |
|---|---|
| **the ball / ShaqBall** | The player-controlled sphere, textured with Shaq's face. Radius 0.5 units, mass 1.0 kg. |
| **the stage** | The entire tilting platform (`stageGroup`) the ball rolls on. 20 units wide (X: -10 to +10), 100 units long (Z: -10 to +90). |
| **the ramp** *(Test Lab)* | Concrete ramp in the Test Lab, Z=15–35, rising from Y=0 to Y=5 (~14° slope). Near end buried below floor for a seamless bottom transition. Far end flush with the high platform. |
| **the high platform** *(Test Lab)* | Raised default-material platform at the far end of the ramp. Top surface at Y=5, 20 units long (Z=35–55), 14 units wide, with metal side rails. |
| **prop zone / prop spawn area** *(Test Lab)* | A 15×15 unit square at ground level (Y≈0.05) starting 3 units past the high platform (Z=58–73). Marked with metal corner posts. Props are spawned here via the Test Lab side panel. |
| **props** | Spawnable interactive obstacles placed in the prop zone via the Test Lab panel. Currently: Spinner, Launcher. |
| **warning zone / danger zone** | The translucent red fan-shaped mesh on the floor showing the arc the launcher arm will sweep through. Non-collidable, purely visual. |
| **the flip** | The forward swing phase of the launcher arm state machine (`swing` state). |
| **the toroid / half-pipe** | A `THREE.TorusGeometry` with arc = π (half circle), rendered `DoubleSide` so the inner curved surface is visible. Random XYZ rotation per spawn. |
| **spinner** | Rotating obstacle: metal hub cylinder + 3 tall concrete paddles at 120° intervals. Rotates around its Y axis each frame. |
| **launcher** | Swing-arm obstacle: metal base + concrete arm that sweeps ~153° when triggered. State machine: idle → windup → swing → return → cooldown. |

---

## Architecture Overview

### Scene Graph
- `scene` — root Three.js scene
  - `stageGroup` — the entire tilting stage, pivots around ball contact point each frame
    - Floor mesh (permanent, child index 0)
    - All stage geometry (ramps, patches, obstacles, powerups)
    - Spinner groups
    - Launcher groups
  - `ballMesh` — Shaq-textured sphere, positioned in world space
  - `sun` — directional light tracking ball position for shadows
  - `gridHelper` — fixed world-space reference grid at Y=-10

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
| Gravity | 25 m/s² |
| Max tilt | 15° |
| Tilt speed | 80 °/s |
| Tilt return speed | 120 °/s |
| Jump speed (base) | 9.6 m/s |
| Death threshold | Y < -15 |

### Collision
Per-triangle brute-force sphere vs mesh test (`sphereMeshCollision`). Every frame, `stageGroup.traverse` tests all meshes with `collidable !== false`. Finds deepest penetrating triangle, returns `{normal, depth}`. Ball position is depenetrated: `pos += n * depth`. `onFloor = true` when `n.y > 0.3`.

### Bounce Response
Normal surfaces: `vel += n * -vn * (1 + restitution)`, clamped to prevent re-penetration. Bounce pads bypass restitution and apply preset `bounceVel = 25` upward when `n.y > 0.5`.

### Surface Velocity (Props)
When the ball contacts a spinner paddle or launcher arm, the world-space velocity of the contact point on the rotating body is computed as `ω × r_contact`. The impulse is applied in the relative velocity frame so the ball is genuinely launched rather than just deflected.

### Friction
Impulse-based. Slip velocity = (linear vel + ω×r) projected onto contact plane. Impulse capped at `μ * Fn * dt`. Applied to both `vel` and `angVel`.

### Rolling Resistance
Braking torque on `angVel` proportional to `Crr * Fn * r * dt / I`, capped at current `|ω|`.

### Jump Buffer
Space press buffered for 200ms — pressing space in the air triggers jump on next landing within the window. One jump per key press (rising edge detection, no OS key-repeat re-trigger).

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

---

## Powerups

Spawn as bobbing/rotating spheres in play mode (2–3 per level). Not present in Test Lab as pickups — toggled live via the Test Lab side panel instead.

| Powerup | Color | Effect |
|---|---|---|
| Size Up | Yellow `#f0c040` | Ball radius ×1.5, jump speed ×1.5 |
| Bouncy Ball | Cyan `#40e0ff` | Restitution multiplier ×1.6 (capped 0.99) |
| Speed Up | Orange-red `#ff6040` | Gravity ×1.8 (faster roll) |
| Jetpack | Green `#80ff80` | Space in air applies 18 m/s² thrust. Fuel drains at 0.45/s, refills at 0.4/s after 2s on ground. |

---

## Obstacles

### Spinners
Rotating `THREE.Group` — metal hub cylinder + 3 tall concrete paddles at 120° intervals. `group.rotation.y += speed * dt` each frame. Scale 0.8–1.4×, speed 0.8–2.5 rad/s, random CW/CCW. 1–3 per play mode level.

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
| windup | 0.8s | Arm flashes orange, no movement |
| swing | until arc complete | Arm sweeps 153° at 5.04 rad/s |
| return | until back at rest | Arm returns at 1.5 rad/s |
| cooldown | 1.0s | Locked, then back to idle |

**Trigger / Warning Zone:**
- Both use the same sector: `innerR = 0.3×scale`, `outerR = 5.0×scale`, arc = 153°
- Trigger converts ball to group-local space and checks radius + angular bounds
- Warning zone is a translucent red (`0xff2200`, opacity 0.22) fan mesh at floor level
- Fan angle: `π/2 - LAUNCHER_SWING_ARC` → `π/2` in group-local XZ (matches actual arm sweep)

### Half-Torus (Toroid / Half-pipe)
`THREE.TorusGeometry` with arc = π. `DoubleSide` material so inner surface renders. Randomised major radius (2–4), tube radius (0.5–1), full random XYZ rotation. 60% spawn probability per play mode level.

### Ramps
Thin tilted `BoxGeometry` slabs (wood/metal/concrete). Random width, length, tilt, and Z position. Spawn from Z=10 onward.

### Bumpers
Tall cylinders (rubber/metal/concrete) acting as fixed posts. Random radius and height.

### Surface Patches
- **Sand patches** — high rolling resistance zones, max 8 units wide (always a gap to squeeze through)
- **Ice patches** — very low friction zones

### Bounce Pads
Flat red cylinders on the floor. Contact when `n.y > 0.5` launches ball upward at 25 m/s regardless of approach velocity.

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

**Props** — each click spawns one prop at a random position within the prop zone. Props accumulate on each click. No level rebuild needed — props are added directly to `stageGroup`.
- `+ Spinner` — random scale (0.8–1.2×), random speed and direction
- `+ Launcher` — random yaw orientation
- `Clear props` — removes all spawned props instantly

---

## Camera

Spherical orbit around ball. Arrow left/right orbit azimuth, arrow up/down snap polar angle. Lerps to target at 8% per frame. Default: azimuth = π (behind ball), polar = mid angle.

---

## Visual Systems

### Shadows
Ball-tracking directional shadow. Sun position and target follow ball each frame. 2048×2048 PCFSoft shadow map, ±30 unit frustum. Edge hidden inside fog.

### Fog
`THREE.Fog` matched to background `#333333`. Starts at 30 units, full at 70. Hides stage edges and shadow frustum boundary.

### Reference Grid
Fixed world-space `GridHelper` at Y=-10, 200×200 units. Does not tilt with stage.

### HUD
Top-right: active powerup indicators. Bottom-center: jetpack fuel bar (visible only when jetpack active).

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

---

## Roadmap

### Planned Features

#### Stage-Locked Physics Objects
Free objects whose physics frame is locked to the stage — move with stage but can still roll/slide on it. Requires running physics in stage-local coordinates each frame. Enables: loose barrels, rolling balls, sliding boxes.

#### Luge / Half-pipe Channel
U-shaped channel running along Z axis. Built from angled box walls meeting at a flat floor. Ball naturally rolls to center under tilt. Interesting because left/right tilt behaves very differently inside vs outside.

#### Funnels
Four angled box faces arranged in a square pyramid, guiding ball toward center. No hole required.

#### Goal Object + Level Clear
Distinct mesh at Z=90. Entering its trigger radius clears the level: flash → difficulty++ → rebuild → ball reset.

#### Level Progression System
Discrete levels via `buildLevel(difficulty)`. Scaling:
- Obstacle count and speed increase with difficulty
- Stage length: starts 100 units, grows ~20 per level, capped ~400
- Surface mix: early = default/wood, late = more ice/sand
- Powerup frequency decreases with difficulty
- Z-axis height variation unlocks at difficulty 3+

#### Z-axis Height Variation
Floor built from segmented `BoxGeometry` panels at varying Y offsets — hills, dips, steps. Biggest visual and gameplay change. Do last when rest is stable.
