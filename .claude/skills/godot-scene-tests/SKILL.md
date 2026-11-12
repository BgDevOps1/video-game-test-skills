---
name: godot-scene-tests
description: Use when testing Godot 4 scenes and node scripts (the game/ folder: hero, enemies, touch controls, HUD) with headless GUT, including deterministic physics stepping, multi-touch input, layout at several aspect ratios and injected game time.
---

# Godot scene tests (GUT, headless)

Rules (`res://rules/`) are tested per `godot-rule-tests`. Anything that needs nodes, the scene tree or input lives in `res://game/` and is tested here. Same runner, same CI job. Put tests in `tests/game/test_<unit>.gd`.

## Instantiate scenes

```gdscript
extends GutTest

const HeroScene := preload("res://game/hero/hero.tscn")
var hero: Hero

func before_each() -> void:
    hero = HeroScene.instantiate()
    add_child_autofree(hero)   # in the tree (so _ready runs) and freed after the test
```

- Always `add_child_autofree`, never a bare `add_child`; leaked nodes bleed between tests.
- Helper nodes (dummy enemies, markers) are created and added the same way. A bare `Node2D` in a group is enough for a collaborator.
- Set `global_position` after adding to the tree.

## Step physics directly

Do not wait on real frames; call the callback with a fixed delta.

```gdscript
const DT := 1.0 / 60.0
func _step(n: int) -> void:
    for i in n:
        hero._physics_process(DT)
```

- Deterministic and instant. Use `await wait_physics_frames(n)` only when the engine itself must do the work (real collisions, Area2D overlaps).
- Derive step counts from tuning values (`int(tuning.dodge_duration / DT) + 2`), not magic numbers.

## Simulate multi-touch

Build `InputEventScreenTouch` / `InputEventScreenDrag` with an explicit `index`, and feed them straight to the node: `controls._input(ev)`.

```gdscript
func _touch(idx: int, pos: Vector2, pressed: bool) -> InputEventScreenTouch:
    var ev := InputEventScreenTouch.new()
    ev.index = idx; ev.position = pos; ev.pressed = pressed
    return ev
```

- Two-thumb test: index 0 presses and drags on the left half, index 1 presses a button; assert both a non-zero move and the button signal.
- Also assert releasing one index leaves the other control working, and that touches on the wrong half are ignored.
- Connect signals with lambdas that append to arrays, then assert on the arrays.

## Layout at several aspect ratios

Expose `layout(viewport_size: Vector2)` on the UI node and call it directly, so no real window resize is needed.

```gdscript
for size in [Vector2(1560, 720), Vector2(960, 720)]:   # 19.5:9 and 4:3
    controls.layout(size)
    for name in controls.button_rects:
        assert_true(Rect2(Vector2.ZERO, size).encloses(controls.button_rects[name]))
```

Expose rects as data (a dictionary), not only as drawn pixels. Draw positions that depend on state (a joystick knob) go through a small function such as `knob_position()` so they can be asserted.

## Inject the game clock

Gameplay methods take time as a parameter (`hero.attack(now)`); the owner (`main.gd`) accumulates `delta` into `game_time`. Never read `Time.get_ticks_msec()` in gameplay code. Tests then pass explicit times (`attack(0.0)`, `attack(0.2)`, `attack(0.4)`) and get exact rhythm results.

## Run and verify

Same commands and red-then-green rule as `godot-rule-tests`. CI greps the log for parse and load errors, which matters here: a scene script with a typo is reported as "Ignoring script", not as a failure.
