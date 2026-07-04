# Mario-style Platformer — Design Spec

Date: 2026-06-16
Status: Approved

## Overview

A sidescrolling platformer in the style of classic Mario, built as a fresh
Summer Engine (Godot, GDScript) 2D project in the `Mario` folder. Square tiles
on a 32×32 px grid, jumping with collision, an x-following / y-locked camera,
ASCII-authored levels, a 3-life heart HUD, a door that advances to the next
level, and basic patrolling goomba enemies.

Visuals use simple generated pixel-art sprites sized to the 32×32 tile grid.

## Goals (v1 scope)

- Sidescrolling platformer movement: run, gravity, jump, tile collision.
- Camera follows the player on the x-axis, locked on the y-axis, clamped to
  level bounds.
- Levels built from a grid of square tiles, authored as ASCII text.
- Reach a door to complete a level and load the next one; win screen after the
  last level.
- 3 lives shown as hearts in the upper-left corner.
- Basic goomba enemies that patrol and damage the player on contact.
- Ship with 3 short levels demonstrating progression.

## Non-goals (v1)

- Stomping/defeating enemies (goombas are pure hazards — contact damages).
- Power-ups, coins, scoring, multiple enemy types.
- Moving platforms, slopes, or non-square geometry.
- In-engine visual TileMap painting as the authoring path (ASCII is the source
  of truth).
- Audio/music (can be added later).

## Tech

- Engine: Summer Engine (Godot-based), GDScript.
- 2D, tile grid of 32×32 px square tiles.
- Generated pixel-art sprites for player, goomba, brick/block, and door.
- Project created fresh in `Mario/`. The user must open this project in Summer
  Engine before it can be played/tested (the engine/MCP only acts on the
  currently open project and cannot switch projects itself).

## Components

### GameManager (autoload singleton)
The single owner of global state.
- State: current level index, lives (start 3), ordered list of level files.
- Responsibilities: load a level by index, advance to the next level on door
  reach, show win screen after the last level, handle losing a life (respawn
  vs game over), reset on game over.
- Interface (conceptual): `start_game()`, `load_level(index)`,
  `next_level()`, `lose_life()`, `game_over()`, `win()`.
- Depends on: LevelLoader, HUD, the level file list.

### LevelLoader
Turns an ASCII level into a live scene.
- Input: an ASCII level (text resource) + the GameManager.
- Builds solid blocks into a single `TileMap` for efficient collision.
- Instances Player, Goomba(s), and Door scenes at their grid cells.
- Computes level pixel bounds (used for camera clamping and fall-off
  threshold).
- Depends on: TileMap/TileSet, Player/Goomba/Door scenes.

### Player (CharacterBody2D)
- Gravity, left/right run, jump; movement/collision via `move_and_slide`.
- Reports events to GameManager: fell off the bottom, touched a goomba,
  reached the door.
- Input: left, right, jump (mapped via the project input map).
- Depends on: input map, GameManager.

### Goomba (CharacterBody2D)
- Patrols horizontally at a fixed speed.
- Turns around when it hits a wall or reaches a ledge.
- Contact with the player damages the player (no stomping).
- Depends on: tile collision, Player detection.

### Door (Area2D)
- Player overlap triggers "level complete → next level" via GameManager.

### HUD (CanvasLayer)
- Displays 3 hearts in the upper-left corner.
- Updates whenever lives change (driven by GameManager).

### Camera2D
- Follows the player on x; y is fixed (locked vertically).
- Clamped to level bounds so it does not scroll past level edges.

## Level format (ASCII text)

Each level is a `.txt` grid file. One character per 32×32 cell:

- `#` = solid block
- `P` = player start
- `G` = goomba
- `D` = door
- `.` or space = empty

Example:

```
................
................
......G.....D...
............#...
P.........#####.
################
```

The LevelLoader parses top-to-bottom, left-to-right; row/column index maps to
world position via the 32px tile size. v1 ships 3 short level files in an
ordered list owned by GameManager.

## Game flow & rules

- Reach the door → load the next level. After the last level → win screen.
- Touching a goomba → lose 1 life.
- Falling off the bottom of the level → lose 1 life.
- Losing a life that is not the last → respawn at the start of the current
  level (level progress index unchanged).
- Losing all 3 lives → game over screen → restart from level 1 with lives
  reset to 3.

## Testing

The agent does NOT run or play the game. After each piece is built, the agent
asks the user to run the relevant scene in Summer Engine and report back the
result (console output + script/debugger errors, and observed behavior). The
agent iterates based on that feedback until it launches clean.

Full-flow manual playtest (move, jump, collide, take damage, respawn, reach
door, advance, game over, win) is performed by the user before the work is
declared done.

The user opens the Mario project in Summer Engine and runs it; the agent reads
the reported diagnostics to verify.

## Open defaults (easy to change)

- Tile size: 32 px.
- Number of v1 levels: 3.
- Game over returns to level 1 (vs. current level).
