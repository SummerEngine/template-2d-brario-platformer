# Dynamic Platforms — Design

Date: 2026-06-16

## Goal

Add moving platforms, slanted platforms, and bonus dynamic platform types to the
Mario platformer, woven into the existing levels (`level_01..03`).

## Existing architecture (baseline)

- **Levels**: ASCII `.txt` files in `levels/`, 32px tiles, parsed by
  `scripts/level_loader.gd` into instanced scenes under `Entities`.
  - Current chars: `#` Block, `P` Player, `G` Goomba, `D` Door, `.` empty.
- **Platforms**: `Block.tscn` = `StaticBody2D`, collision_layer 1, mask 0, 32×32
  rectangle. Sprite applied at runtime via `SpriteUtil.apply(...)`.
- **Player**: `Player.tscn` = `CharacterBody2D`, layer 2, mask 1, 28×46 box.
  `player.gd` — gravity 1100, jump -560, speed 200, `move_and_slide()`.
  **No slope settings, no moving-platform handling.**
- **Collision layers**: 1=world, 2=player, 4=enemy, 8=door.
- Godot 4.6, GDScript 2.0.

## New platform types

All new solid platforms live on **collision layer 1** so existing player collision
(mask 1) and velocity inheritance work without changing layer conventions.

| Char | Type            | Node                | Behavior |
|------|-----------------|---------------------|----------|
| `-`  | Horizontal mover| `AnimatableBody2D`  | ping-pongs left↔right, sine motion |
| `\|` | Vertical mover  | `AnimatableBody2D`  | ping-pongs up↕down (elevator) |
| `/`  | Slope up        | `StaticBody2D`      | triangular `CollisionPolygon2D`, rises L→R |
| `\`  | Slope down      | `StaticBody2D`      | mirrored triangle |
| `o`  | Bounce pad      | `StaticBody2D`+`Area2D` | super-jump on land |
| `x`  | Crumble block   | `StaticBody2D`+`Area2D` | falls away ~0.4s after player stands, respawns |

### Moving platforms
- `AnimatableBody2D`, `sync_to_physics = true` (auto-computes velocity from motion
  so CharacterBody2D inherits it).
- `scripts/moving_platform.gd` with `@export axis: Vector2`, `@export distance`,
  `@export speed`. `_physics_process`: `t += delta`; position = origin + axis *
  sin(t*speed) * distance. (No `Date`/`randf` — internal accumulator only.)
- Two preset scenes: `MovingPlatformH.tscn` (axis 1,0) and `MovingPlatformV.tscn`
  (axis 0,1). Visual ~48×16 via SpriteUtil placeholder.

### Slopes
- `StaticBody2D` (layer 1) with a triangular `CollisionPolygon2D` inside a 32×32
  cell, plus a `Polygon2D` of the same points for the visual (SpriteUtil makes
  rectangles, which can't show a triangle).
- SlopeUp points: `(-16,16),(16,16),(16,-16)`. SlopeDown: `(-16,16),(16,16),(-16,-16)`.

### Bounce pad
- Solid `StaticBody2D` (layer 1) + child `Area2D` "BounceZone" on top
  (collision_layer 0, mask 2 → detects player). On `body_entered`, call
  `body.bounce(BOUNCE_VELOCITY)` if the method exists.
- `player.gd` gains `func bounce(strength)`.

### Crumble block
- Solid `StaticBody2D` (layer 1) + child `Area2D` top detector (layer 0, mask 2).
- On player overlap: shake, after ~0.4s disable collision + fade out; restore
  after ~3s. Self-contained (no player method needed).

## Player controller changes (`player.gd` / `Player.tscn`)

- `Player.tscn`: `floor_max_angle ≈ 0.873` (50°), `floor_snap_length ≈ 10`.
- `player.gd`: add `func bounce(strength: float)`; verify slope walk + mover ride.
- Keep `platform_floor_layers` default (all) so layer-1 movers carry the player.

## Level loader changes (`scripts/level_loader.gd`)

- Extend the char→PackedScene map with the six new chars → the six new scenes.
- Movers/slopes/crumble/bounce are pure char→scene instances at cell center;
  no per-instance config (presets baked into the scenes).

## Level edits

- Weave new chars into `level_01.txt`, `level_02.txt`, `level_03.txt` in spots with
  enough clearance for movers to oscillate and slopes to connect ledges. Keep each
  level completable.

## Out of scope

- No new art assets (placeholder colors / polygons only).
- No new input actions. No HUD changes.

## Test

Engine is not driven by the agent — user playtests after build (walk up/down
slopes, ride both movers, bounce off the pad, fall through a crumble block).
