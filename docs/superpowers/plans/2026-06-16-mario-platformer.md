# Mario-style Platformer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a sidescrolling Mario-style platformer in Summer Engine (Godot 4.6, GDScript): run/jump/collide on square tiles, x-following / y-locked camera, ASCII-authored levels, a door that advances levels, a 3-heart lives HUD, and patrolling goomba enemies.

**Architecture:** A `GameManager` autoload owns persistent state (lives, current level index, level file list) and drives flow via Godot scene reloads — because the autoload survives scene changes, "respawn" and "next level" are just `reload_current_scene()`, and the `Level` scene rebuilds itself from the current index on `_ready`. The `Level` scene's `level_loader.gd` parses an ASCII text level into a `TileMapLayer` (one merged collision body for solid blocks) plus instanced `Player`, `Goomba`, and `Door` scenes. Visuals start as solid-color placeholders generated in code and are swapped for generated pixel-art sprites in the final task (no scene edits needed — a shared `SpriteUtil` loads the real PNG if present, else fills a solid color).

**Tech Stack:** Godot 4.6 / GDScript, Summer Engine MCP (for asset generation + running). No automated test framework — verification is performed by the USER running scenes in Summer Engine and reporting console output / diagnostics. The agent does NOT run the game.

---

## Verification model (read first)

This project has no unit-test runner. For every task, the "test" steps mean:

1. The agent writes the files.
2. The agent asks the **user** to open the relevant scene in Summer Engine and press Play, then report back: (a) any console / debugger errors, (b) the observed on-screen behavior.
3. The agent iterates based on that report until the "Expected" behavior is met.

The agent MUST NOT call `summer_play` / run the game itself. Asset generation (`summer_generate_image`) and import DO use the live engine and require the Mario project to be **open** in Summer Engine.

Git is optional for this project (it is not currently a git repo). Commit steps below are suggested checkpoints; do them only if the user has enabled git. To enable: `git init` in `Mario/` during Task 1.

---

## File structure

```
Mario/
  project.godot                  # config, autoload, input map, window/2D settings
  icon.svg                       # default Godot icon (optional)
  scenes/
    Level.tscn                   # main scene: Terrain + Entities + Camera2D + HUD
    Player.tscn                  # CharacterBody2D
    Goomba.tscn                  # CharacterBody2D + Hitbox Area2D + ledge rays
    Door.tscn                    # Area2D
    HUD.tscn                     # CanvasLayer with 3 heart TextureRects
    GameOver.tscn                # CanvasLayer game-over screen
    WinScreen.tscn               # CanvasLayer win screen
  scripts/
    game_manager.gd              # autoload singleton (state + flow)
    level_loader.gd              # parses ASCII -> tilemap + entities (Level root)
    sprite_util.gd               # SpriteUtil.texture_for() helper
    camera.gd                    # x-follow, y-locked, clamped Camera2D
    player.gd
    goomba.gd
    door.gd
    hud.gd
    screen.gd                    # shared "press any key to continue" screen
  levels/
    level_01.txt
    level_02.txt
    level_03.txt
  assets/
    sprites/                     # generated PNGs land here in Task 8
  docs/superpowers/...           # spec + this plan
```

---

## Task 1: Project scaffold

**Files:**
- Create: `Mario/project.godot`
- Create: `Mario/scripts/game_manager.gd` (stub — full version in Task 2)
- Create: `Mario/scenes/Level.tscn` (placeholder — full version in Task 2)
- Create: `Mario/levels/level_01.txt` (placeholder — full version in Task 7)
- Create dirs: `Mario/assets/sprites/`

- [ ] **Step 1: Write `project.godot`**

```ini
; Engine configuration file.
config_version=5

[application]

config/name="Mario"
config/description="A sidescrolling Mario-style platformer built for Summer Engine."
run/main_scene="res://scenes/Level.tscn"
config/features=PackedStringArray("4.6")

[autoload]

GameManager="*res://scripts/game_manager.gd"

[display]

window/size/viewport_width=640
window/size/viewport_height=360
window/stretch/mode="canvas_items"
window/stretch/aspect="keep"

[input]

move_left={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":65,"key_label":0,"unicode":97,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194319,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
move_right={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":68,"key_label":0,"unicode":100,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194321,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
jump={
"deadzone": 0.5,
"events": [Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":32,"key_label":0,"unicode":32,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":87,"key_label":0,"unicode":119,"location":0,"echo":false,"script":null)
, Object(InputEventKey,"resource_local_to_scene":false,"resource_name":"","device":-1,"window_id":0,"alt_pressed":false,"shift_pressed":false,"ctrl_pressed":false,"meta_pressed":false,"pressed":false,"keycode":0,"physical_keycode":4194320,"key_label":0,"unicode":0,"location":0,"echo":false,"script":null)
]
}
```

(move_left = A / Left arrow; move_right = D / Right arrow; jump = Space / W / Up arrow.)

- [ ] **Step 2: Write a temporary `scripts/game_manager.gd` stub** (so autoload resolves)

```gdscript
extends Node
# Stub — replaced in Task 2.
var lives := 3
var current_level := 0
```

- [ ] **Step 3: Write a temporary `scenes/Level.tscn`** (so the main scene resolves)

```
[gd_scene format=3]

[node name="Level" type="Node2D"]

[node name="Label" type="Label" parent="."]
offset_left = 200.0
offset_top = 160.0
text = "Mario scaffold OK"
```

- [ ] **Step 4: Create `levels/level_01.txt`** (placeholder one-liner; real levels in Task 7)

```
P......D
########
```

- [ ] **Step 5: Create the `assets/sprites/` directory** (empty, with a `.gdignore`-free placeholder)

Create file `Mario/assets/sprites/.keep` with content `placeholder`.

- [ ] **Step 6: (Optional) init git**

```bash
cd Mario && git init && git add -A && git commit -m "chore: scaffold Mario Godot project"
```

- [ ] **Step 7: HALT and ask the user to open the project**

Tell the user: "Scaffold written. Please **open the `Mario` folder as a project in Summer Engine** (the engine currently has the Metroidvania template open and can't switch projects itself). Then press Play on the main scene and tell me: does a window open showing the text 'Mario scaffold OK' with no errors in the console?"

Expected: a 640×360 window opens, shows the label, no script/parse errors. If the project won't import, fix `project.godot` based on the reported error before continuing.

---

## Task 2: Core — autoload, sprite helper, level loader (blocks only), camera

This task gets a solid-tile level rendering with a working camera, no player yet.

**Files:**
- Create: `Mario/scripts/sprite_util.gd`
- Modify (replace stub): `Mario/scripts/game_manager.gd`
- Create: `Mario/scripts/camera.gd`
- Create: `Mario/scripts/level_loader.gd`
- Modify (replace placeholder): `Mario/scenes/Level.tscn`

- [ ] **Step 1: Write `scripts/sprite_util.gd`**

```gdscript
class_name SpriteUtil
extends RefCounted

# Returns the real texture at `path` if it exists in the project,
# otherwise a solid-color `size`x`size` placeholder. This lets every
# entity render before generated art exists, and automatically pick up
# the real PNG once it is added (Task 8) with no scene edits.
static func texture_for(path: String, color: Color, size: int) -> Texture2D:
	if ResourceLoader.exists(path):
		var t := load(path)
		if t is Texture2D:
			return t
	var img := Image.create(size, size, false, Image.FORMAT_RGBA8)
	img.fill(color)
	return ImageTexture.create_from_image(img)
```

- [ ] **Step 2: Write `scripts/game_manager.gd`** (full version, replacing the stub)

```gdscript
extends Node
# Autoload singleton. Persists across scene reloads, so respawn / next-level
# are just reload_current_scene(); the Level scene rebuilds from current_level.

signal lives_changed(lives: int)

const MAX_LIVES := 3

var level_files := [
	"res://levels/level_01.txt",
	"res://levels/level_02.txt",
	"res://levels/level_03.txt",
]
var current_level := 0
var lives := MAX_LIVES

func get_current_level_path() -> String:
	return level_files[current_level]

# Called by Player when it dies (goomba contact or fell off the bottom).
func player_died() -> void:
	lives -= 1
	lives_changed.emit(lives)
	if lives <= 0:
		get_tree().change_scene_to_file("res://scenes/GameOver.tscn")
	else:
		get_tree().reload_current_scene()

# Called by Door when the player reaches it.
func next_level() -> void:
	current_level += 1
	if current_level >= level_files.size():
		get_tree().change_scene_to_file("res://scenes/WinScreen.tscn")
	else:
		get_tree().reload_current_scene()

# Called by the GameOver / WinScreen continue action.
func reset_game() -> void:
	current_level = 0
	lives = MAX_LIVES
	lives_changed.emit(lives)
	get_tree().change_scene_to_file("res://scenes/Level.tscn")
```

- [ ] **Step 3: Write `scripts/camera.gd`**

```gdscript
extends Camera2D
# Follows `target` on x, locked on y, clamped to level bounds.

var target: Node2D = null
var min_x := 0.0
var max_x := 0.0
var fixed_y := 0.0

func _process(_delta: float) -> void:
	if target == null:
		return
	var x := clampf(target.global_position.x, min_x, maxf(min_x, max_x))
	global_position = Vector2(x, fixed_y)
```

- [ ] **Step 4: Write `scripts/level_loader.gd`** (blocks + camera; player/goomba/door wired but their scenes are added in later tasks via preload — include the full version now so later tasks only add the scene files)

```gdscript
extends Node2D
# Parses the current ASCII level into a TileMapLayer of solid blocks plus
# instanced Player / Goomba / Door scenes, then configures the camera.

const TILE := 32
const VIEW_HALF_W := 320.0
const VIEW_HALF_H := 180.0

@onready var terrain: TileMapLayer = $Terrain
@onready var entities: Node2D = $Entities
@onready var camera = $Camera2D

func _ready() -> void:
	var path := GameManager.get_current_level_path()
	var text := ""
	if FileAccess.file_exists(path):
		text = FileAccess.get_file_as_string(path)
	else:
		push_error("Level file not found: " + path)
	_build(text)

func _build(text: String) -> void:
	terrain.tile_set = _make_tileset()

	var lines := text.replace("\r", "").split("\n")
	var rows := lines.size()
	var cols := 0
	var player: Node2D = null

	for y in range(rows):
		var line: String = lines[y]
		cols = maxi(cols, line.length())
		for x in range(line.length()):
			var c := line[x]
			var pos := Vector2(x * TILE + TILE / 2.0, y * TILE + TILE / 2.0)
			match c:
				"#":
					terrain.set_cell(Vector2i(x, y), 0, Vector2i(0, 0))
				"P":
					player = _spawn("res://scenes/Player.tscn", pos)
				"G":
					_spawn("res://scenes/Goomba.tscn", pos)
				"D":
					_spawn("res://scenes/Door.tscn", pos)
				_:
					pass

	var level_w := cols * TILE
	var level_h := rows * TILE

	if player != null and "kill_y" in player:
		player.kill_y = level_h + 200.0

	camera.target = player
	camera.min_x = VIEW_HALF_W
	camera.max_x = maxf(VIEW_HALF_W, level_w - VIEW_HALF_W)
	camera.fixed_y = maxf(VIEW_HALF_H, level_h - VIEW_HALF_H)

func _spawn(scene_path: String, pos: Vector2) -> Node2D:
	if not ResourceLoader.exists(scene_path):
		return null
	var inst := load(scene_path).instantiate()
	inst.position = pos
	entities.add_child(inst)
	return inst

# Builds a TileSet at runtime: one tile, one physics layer on collision
# layer 1, with a 32x32 box collider. Texture is the brick sprite if it
# exists, else a solid brown placeholder.
func _make_tileset() -> TileSet:
	var ts := TileSet.new()
	ts.tile_size = Vector2i(TILE, TILE)
	ts.add_physics_layer()
	ts.set_physics_layer_collision_layer(0, 1)
	ts.set_physics_layer_collision_mask(0, 0)

	var src := TileSetAtlasSource.new()
	src.texture = SpriteUtil.texture_for("res://assets/sprites/brick.png", Color(0.55, 0.35, 0.2), TILE)
	src.texture_region_size = Vector2i(TILE, TILE)
	src.create_tile(Vector2i(0, 0))

	var data := src.get_tile_data(Vector2i(0, 0), 0)
	var half := TILE / 2.0
	data.add_collision_polygon(0)
	data.set_collision_polygon_points(0, 0, PackedVector2Array([
		Vector2(-half, -half), Vector2(half, -half),
		Vector2(half, half), Vector2(-half, half),
	]))

	ts.add_source(src, 0)
	return ts
```

- [ ] **Step 5: Write `scenes/Level.tscn`** (replace placeholder)

```
[gd_scene load_steps=4 format=3]

[ext_resource type="Script" path="res://scripts/level_loader.gd" id="1"]
[ext_resource type="Script" path="res://scripts/camera.gd" id="2"]
[ext_resource type="PackedScene" path="res://scenes/HUD.tscn" id="3"]

[node name="Level" type="Node2D"]
script = ExtResource("1")

[node name="Terrain" type="TileMapLayer" parent="."]

[node name="Entities" type="Node2D" parent="."]

[node name="Camera2D" type="Camera2D" parent="."]
script = ExtResource("2")

[node name="HUD" parent="." instance=ExtResource("3")]
```

NOTE: `HUD.tscn` does not exist until Task 6. Until then, temporarily remove the `id="3"` ext_resource line AND the `[node name="HUD" ...]` line from `Level.tscn`, and re-add them in Task 6. (The loader does not reference HUD, so it runs fine without it.) For this task, use this trimmed scene instead:

```
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/level_loader.gd" id="1"]
[ext_resource type="Script" path="res://scripts/camera.gd" id="2"]

[node name="Level" type="Node2D"]
script = ExtResource("1")

[node name="Terrain" type="TileMapLayer" parent="."]

[node name="Entities" type="Node2D" parent="."]

[node name="Camera2D" type="Camera2D" parent="."]
script = ExtResource("2")
```

- [ ] **Step 6: Give Task-2 a testable level** — overwrite `levels/level_01.txt` with a block layout (no entities yet; loader skips unknown chars):

```
................................
................................
................................
................................
................................
................................
................................
............########............
................................
................................
################################
################################
```

- [ ] **Step 7: Ask the user to run `Level.tscn`**

Expected report: window opens; a brown floor (two rows) spans the bottom and a brown platform floats in the middle; no errors. The camera will sit still (no target/player yet) — that's fine. If you see "Invalid call" on TileSet methods, the Godot version differs from 4.6 — report the exact error so the agent can adjust TileSet API calls.

- [ ] **Step 8: (Optional) commit**

```bash
cd Mario && git add -A && git commit -m "feat: level loader builds tilemap + camera"
```

---

## Task 3: Player — move, jump, collide, fall-death, camera follow

**Files:**
- Create: `Mario/scripts/player.gd`
- Create: `Mario/scenes/Player.tscn`
- Modify: `Mario/levels/level_01.txt` (add a `P`)

- [ ] **Step 1: Write `scripts/player.gd`**

```gdscript
extends CharacterBody2D

const SPEED := 200.0
const JUMP_VELOCITY := -430.0
const GRAVITY := 1100.0

var kill_y := 100000.0
var dead := false

@onready var sprite: Sprite2D = $Sprite2D

func _ready() -> void:
	add_to_group("player")
	sprite.texture = SpriteUtil.texture_for("res://assets/sprites/player.png", Color(0.2, 0.4, 0.9), 32)

func _physics_process(delta: float) -> void:
	if dead:
		return

	if not is_on_floor():
		velocity.y += GRAVITY * delta

	if Input.is_action_just_pressed("jump") and is_on_floor():
		velocity.y = JUMP_VELOCITY

	var dir := Input.get_axis("move_left", "move_right")
	velocity.x = dir * SPEED
	if dir != 0.0:
		sprite.flip_h = dir < 0.0

	move_and_slide()

	if global_position.y > kill_y:
		die()

func die() -> void:
	if dead:
		return
	dead = true
	GameManager.player_died()
```

- [ ] **Step 2: Write `scenes/Player.tscn`**

```
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/player.gd" id="1"]

[sub_resource type="RectangleShape2D" id="shape"]
size = Vector2(28, 32)

[node name="Player" type="CharacterBody2D"]
collision_layer = 2
collision_mask = 1
script = ExtResource("1")

[node name="Sprite2D" type="Sprite2D" parent="."]

[node name="CollisionShape2D" type="CollisionShape2D" parent="."]
shape = SubResource("shape")
```

- [ ] **Step 3: Add a player start to `levels/level_01.txt`** (put `P` on the floor area, left side)

```
................................
................................
................................
................................
................................
................................
................................
............########............
................................
..P.............................
################################
################################
```

- [ ] **Step 4: Ask the user to run `Level.tscn`**

Expected report: a blue square (player) falls onto the floor; **A/D or arrows** move left/right; **Space/W/Up** jumps (~2 tiles high); the camera follows horizontally but stays vertically fixed; walking off the right edge into open air and falling below the level reloads the level (player reappears at start). No errors. If movement feels too floaty/stiff or the jump can't clear the floating platform, report it and the agent will tune `SPEED` / `JUMP_VELOCITY` / `GRAVITY`.

- [ ] **Step 5: (Optional) commit**

```bash
cd Mario && git add -A && git commit -m "feat: player movement, jump, collision, fall death"
```

---

## Task 4: Door — advance to next level

**Files:**
- Create: `Mario/scripts/door.gd`
- Create: `Mario/scenes/Door.tscn`
- Modify: `Mario/levels/level_01.txt` (add a `D`)

- [ ] **Step 1: Write `scripts/door.gd`**

```gdscript
extends Area2D

var used := false

@onready var sprite: Sprite2D = $Sprite2D

func _ready() -> void:
	sprite.texture = SpriteUtil.texture_for("res://assets/sprites/door.png", Color(0.2, 0.7, 0.3), 32)
	body_entered.connect(_on_body_entered)

func _on_body_entered(body: Node) -> void:
	if used:
		return
	if body.is_in_group("player"):
		used = true
		GameManager.next_level()
```

- [ ] **Step 2: Write `scenes/Door.tscn`**

```
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://scripts/door.gd" id="1"]

[sub_resource type="RectangleShape2D" id="shape"]
size = Vector2(28, 32)

[node name="Door" type="Area2D"]
collision_layer = 8
collision_mask = 2
script = ExtResource("1")

[node name="Sprite2D" type="Sprite2D" parent="."]

[node name="CollisionShape2D" type="CollisionShape2D" parent="."]
shape = SubResource("shape")
```

- [ ] **Step 3: Add a door to `levels/level_01.txt`** (right side, on the floor)

```
................................
................................
................................
................................
................................
................................
................................
............########............
................................
..P........................D....
################################
################################
```

- [ ] **Step 4: Temporarily make level 2 distinguishable** so reaching the door visibly does something. Overwrite `levels/level_02.txt`:

```
................................
................................
................................
............D...................
...........####.................
................................
................................
..P.............................
################################
################################
```

- [ ] **Step 5: Ask the user to run `Level.tscn`**

Expected report: walking the player (green square = door) into the door loads level 2 (a visibly different layout — a door on a raised platform). Reaching level 2's door (after the last level) loads `WinScreen.tscn`, which does not exist yet, so expect an error like "Can't load WinScreen.tscn" at that point — that's fine and fixed in Task 6. No other errors. Confirm the level transition works.

- [ ] **Step 6: (Optional) commit**

```bash
cd Mario && git add -A && git commit -m "feat: door advances to next level"
```

---

## Task 5: Goomba — patrol + contact damage

**Files:**
- Create: `Mario/scripts/goomba.gd`
- Create: `Mario/scenes/Goomba.tscn`
- Modify: `Mario/levels/level_01.txt` (add a `G`)

- [ ] **Step 1: Write `scripts/goomba.gd`**

```gdscript
extends CharacterBody2D

const SPEED := 55.0
const GRAVITY := 1100.0

var dir := -1

@onready var sprite: Sprite2D = $Sprite2D
@onready var ledge_left: RayCast2D = $LedgeLeft
@onready var ledge_right: RayCast2D = $LedgeRight

func _ready() -> void:
	sprite.texture = SpriteUtil.texture_for("res://assets/sprites/goomba.png", Color(0.45, 0.22, 0.12), 32)
	$Hitbox.body_entered.connect(_on_hitbox_body_entered)

func _physics_process(delta: float) -> void:
	if not is_on_floor():
		velocity.y += GRAVITY * delta
	else:
		# Turn around at a ledge (no ground ahead in the travel direction).
		var ray := ledge_left if dir < 0 else ledge_right
		if not ray.is_colliding():
			dir *= -1

	velocity.x = dir * SPEED
	move_and_slide()

	# Turn around at a wall.
	if is_on_wall():
		dir *= -1

	sprite.flip_h = dir > 0

func _on_hitbox_body_entered(body: Node) -> void:
	if body.has_method("die"):
		body.die()
```

- [ ] **Step 2: Write `scenes/Goomba.tscn`**

```
[gd_scene load_steps=4 format=3]

[ext_resource type="Script" path="res://scripts/goomba.gd" id="1"]

[sub_resource type="RectangleShape2D" id="body"]
size = Vector2(28, 32)

[sub_resource type="RectangleShape2D" id="hit"]
size = Vector2(26, 30)

[node name="Goomba" type="CharacterBody2D"]
collision_layer = 4
collision_mask = 1
script = ExtResource("1")

[node name="Sprite2D" type="Sprite2D" parent="."]

[node name="CollisionShape2D" type="CollisionShape2D" parent="."]
shape = SubResource("body")

[node name="Hitbox" type="Area2D" parent="."]
collision_layer = 4
collision_mask = 2

[node name="HitboxShape" type="CollisionShape2D" parent="Hitbox"]
shape = SubResource("hit")

[node name="LedgeLeft" type="RayCast2D" parent="."]
position = Vector2(-18, 0)
target_position = Vector2(0, 26)

[node name="LedgeRight" type="RayCast2D" parent="."]
position = Vector2(18, 0)
target_position = Vector2(0, 26)
```

(RayCast2D `enabled` defaults to true. The rays collide with the terrain on physics layer 1 by default.)

- [ ] **Step 3: Add a goomba to `levels/level_01.txt`**

```
................................
................................
................................
................................
................................
................................
................................
............########............
................................
..P.............G..........D....
################################
################################
```

- [ ] **Step 4: Ask the user to run `Level.tscn`**

Expected report: a brown square (goomba) walks back and forth on the floor, reversing at walls and at the floating platform's ledges (it should NOT walk off the floor edge into the pit if there is one; on a full floor it patrols freely and reverses at side walls). Touching the goomba reloads the level (player died). No errors. If the goomba falls through the floor or doesn't reverse at ledges, report it — the agent will adjust the ledge raycast `target_position` / `position`.

- [ ] **Step 5: (Optional) commit**

```bash
cd Mario && git add -A && git commit -m "feat: goomba patrol and contact damage"
```

---

## Task 6: HUD hearts, GameOver and WinScreen, wire HUD into Level

**Files:**
- Create: `Mario/scripts/hud.gd`
- Create: `Mario/scenes/HUD.tscn`
- Create: `Mario/scripts/screen.gd`
- Create: `Mario/scenes/GameOver.tscn`
- Create: `Mario/scenes/WinScreen.tscn`
- Modify: `Mario/scenes/Level.tscn` (re-add the HUD instance removed in Task 2)

- [ ] **Step 1: Write `scripts/hud.gd`**

```gdscript
extends CanvasLayer

@onready var hearts: HBoxContainer = $Margin/Hearts

func _ready() -> void:
	GameManager.lives_changed.connect(_update)
	# Apply the heart texture (real PNG if present, else solid red).
	for child in hearts.get_children():
		if child is TextureRect:
			child.texture = SpriteUtil.texture_for("res://assets/sprites/heart.png", Color(0.9, 0.15, 0.2), 24)
	_update(GameManager.lives)

func _update(lives: int) -> void:
	var i := 0
	for child in hearts.get_children():
		child.visible = i < lives
		i += 1
```

- [ ] **Step 2: Write `scenes/HUD.tscn`** (3 heart TextureRects, top-left)

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/hud.gd" id="1"]

[node name="HUD" type="CanvasLayer"]
script = ExtResource("1")

[node name="Margin" type="MarginContainer" parent="."]
offset_left = 8.0
offset_top = 8.0
theme_override_constants/margin_left = 0

[node name="Hearts" type="HBoxContainer" parent="Margin"]
theme_override_constants/separation = 4

[node name="Heart1" type="TextureRect" parent="Margin/Hearts"]
custom_minimum_size = Vector2(24, 24)

[node name="Heart2" type="TextureRect" parent="Margin/Hearts"]
custom_minimum_size = Vector2(24, 24)

[node name="Heart3" type="TextureRect" parent="Margin/Hearts"]
custom_minimum_size = Vector2(24, 24)
```

- [ ] **Step 3: Write `scripts/screen.gd`** (shared "press any key to continue")

```gdscript
extends CanvasLayer
# Used by GameOver and WinScreen. Any key press restarts the game.

func _unhandled_input(event: InputEvent) -> void:
	if event is InputEventKey and event.pressed and not event.echo:
		GameManager.reset_game()
	elif event is InputEventMouseButton and event.pressed:
		GameManager.reset_game()
```

- [ ] **Step 4: Write `scenes/GameOver.tscn`**

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/screen.gd" id="1"]

[node name="GameOver" type="CanvasLayer"]
script = ExtResource("1")

[node name="Label" type="Label" parent="."]
offset_left = 220.0
offset_top = 150.0
text = "GAME OVER
Press any key to restart"
horizontal_alignment = 1
```

- [ ] **Step 5: Write `scenes/WinScreen.tscn`**

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://scripts/screen.gd" id="1"]

[node name="WinScreen" type="CanvasLayer"]
script = ExtResource("1")

[node name="Label" type="Label" parent="."]
offset_left = 240.0
offset_top = 150.0
text = "YOU WIN!
Press any key to play again"
horizontal_alignment = 1
```

- [ ] **Step 6: Re-add the HUD to `scenes/Level.tscn`** (full version with HUD restored)

```
[gd_scene load_steps=4 format=3]

[ext_resource type="Script" path="res://scripts/level_loader.gd" id="1"]
[ext_resource type="Script" path="res://scripts/camera.gd" id="2"]
[ext_resource type="PackedScene" path="res://scenes/HUD.tscn" id="3"]

[node name="Level" type="Node2D"]
script = ExtResource("1")

[node name="Terrain" type="TileMapLayer" parent="."]

[node name="Entities" type="Node2D" parent="."]

[node name="Camera2D" type="Camera2D" parent="."]
script = ExtResource("2")

[node name="HUD" parent="." instance=ExtResource("3")]
```

- [ ] **Step 7: Ask the user to run `Level.tscn`**

Expected report: three red squares (hearts) in the top-left. Touching a goomba or falling off removes one heart and respawns at the level start; losing the 3rd heart shows the GAME OVER screen; pressing a key restarts from level 1 with 3 hearts. Reaching the door on the final level shows YOU WIN!; a key press restarts. No errors. Confirm heart count decrements correctly and both screens work.

- [ ] **Step 8: (Optional) commit**

```bash
cd Mario && git add -A && git commit -m "feat: HUD hearts, game over and win screens"
```

---

## Task 7: Three real levels

**Files:**
- Modify: `Mario/levels/level_01.txt`
- Modify: `Mario/levels/level_02.txt`
- Create: `Mario/levels/level_03.txt`

Legend: `#` solid block, `P` player start, `G` goomba, `D` door, `.` empty. Tile = 32px. Player jump clears ~2 tiles up / ~4 tiles across, so keep gaps ≤3 tiles and platform steps ≤2 tiles.

- [ ] **Step 1: Write `levels/level_01.txt`** (gentle intro: walk, one jump, one goomba)

```
..........................................
..........................................
..........................................
..........................................
..........................................
..........................................
.....................#####................
..........................................
..P..........G...................G.....D..
..........................................
##########################################
##########################################
```

- [ ] **Step 2: Write `levels/level_02.txt`** (a gap to jump and a raised door)

```
..........................................
..........................................
..........................................
..........................................
...............####.......................
..........................................
.............G.......G.................D..
.................................#####....
..P...........................############
########............######################
##########.......#########################
##########.......#########################
```

(The gap at columns ~10–16 on the bottom rows is 3 tiles wide of pit — jumpable. The door sits on a stepped platform.)

- [ ] **Step 3: Write `levels/level_03.txt`** (more goombas + platform climb)

```
..........................................
..........................................
..........................................
.................................####D....
..........................#####...........
..................####....................
..........####............................
....G.......................G.............
..P.....G..........G..................G...
..........................................
##########################################
##########################################
```

- [ ] **Step 4: Ask the user to play through all three levels start to finish**

Expected report: each level is completable — the player can reach every door without impossible jumps, goombas patrol sanely, and finishing level 3 shows the win screen. Note any jump that's too hard, any goomba that falls off or gets stuck, or any unreachable door. The agent tunes level geometry (and, if needed, `JUMP_VELOCITY`/`SPEED`) based on the report.

- [ ] **Step 5: (Optional) commit**

```bash
cd Mario && git add -A && git commit -m "feat: three playable levels"
```

---

## Task 8: Generate and apply pixel-art sprites

This is the only task that requires generating assets via the live engine. The Mario project MUST be open in Summer Engine.

**Files:**
- Create (generated): `Mario/assets/sprites/player.png`, `goomba.png`, `brick.png`, `door.png`, `heart.png`

No code or scene changes are needed — `SpriteUtil.texture_for()` and `_make_tileset()` already prefer these paths and fall back to solid colors only when they are missing.

- [ ] **Step 1: Generate the five sprites** via `summer_generate_image`, each as 32×32 pixel art, transparent background where appropriate:
  - `player.png` — a small Mario-like platformer hero, front/side facing, 32×32 pixel art, transparent background.
  - `goomba.png` — a small brown mushroom-shaped enemy, 32×32 pixel art, transparent background.
  - `brick.png` — a seamless brown brick block tile, 32×32 pixel art, opaque (tiles edge-to-edge).
  - `door.png` — a closed wooden door / level-exit, 32×32 pixel art, transparent background.
  - `heart.png` — a red pixel-art heart for the lives HUD, 32×32, transparent background.

- [ ] **Step 2: Import/place each generated image** into the project at exactly `res://assets/sprites/<name>.png` (use the engine's asset import flow; resolve the exact MCP call — e.g. `summer_import_asset` / `summer_get_asset_download_url` — against the live project at execution time). Verify each file exists at the expected `res://` path and is imported (a `.import` sidecar is generated).

- [ ] **Step 3: Ask the user to run `Level.tscn`**

Expected report: the player, goombas, bricks, door, and HUD hearts now show the generated pixel art instead of solid colored squares, at the correct sizes, with no broken-texture icons and no errors. If any sprite looks wrong-sized or has a stray background, the agent regenerates just that one.

- [ ] **Step 4: Full final playtest**

Ask the user to play all three levels through to the win screen, confirming art + gameplay together: movement, jumping, collision, goomba damage, heart decrement, respawn, level transitions, game over, and win. This is the completion gate.

- [ ] **Step 5: (Optional) commit**

```bash
cd Mario && git add -A && git commit -m "feat: generated pixel-art sprites"
```

---

## Self-review notes (coverage vs. spec)

- Sidescrolling movement + gravity + jump + tile collision → Task 3 (player) + Task 2 (tilemap collision). ✅
- Camera follows x, locked y, clamped to bounds → Task 2 (`camera.gd`) + loader config. ✅
- Square tiles in a grid, ASCII-authored → Task 2 loader + Task 7 levels. ✅
- Reach door → next level; win screen after last → Task 4 + Task 6. ✅
- 3 lives as hearts top-left → Task 6 HUD. ✅
- Touch goomba = damage (no stomp); fall off = damage → Task 5 + Task 3 `kill_y`. ✅
- Lose a life → respawn at current level start (reload); lose all 3 → game over → level 1 → Task 2 `GameManager` + Task 6. ✅
- Goomba patrol (wall + ledge turn) → Task 5. ✅
- Generated pixel-art sprites → Task 8. ✅
- Agent does not run the game; user verifies → stated in every task's run step. ✅
```
