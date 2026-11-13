---
name: godot-robot-players
description: Use when you need to know how a Godot 4 game plays over many minutes (how fast skills level, when enemies adapt, how often the player dies) without a human playing, or need a test that a whole play session still works.
---

# Godot robot players (scripted bots + balance report)

Load the real main scene, step it by hand at a fixed delta, and let a scripted bot press the buttons. 15 simulated minutes take a few real seconds. The bots are plain code; they call no AI model.

## Files

- `tests/sim/sim_world.gd` (`class_name SimWorld`): loads `main.tscn`, steps it.
- `tests/sim/bot.gd` (`class_name Bot`): walks to the nearest enemy and repeats a script.
- `tests/sim/test_bots.gd`: GUT tests ("within N simulated minutes, X happens").
- `tools/balance_report.gd` + `.tscn`: prints when milestones happen per bot style.
- Add `tools/*` to every export preset's `exclude_filter` (`tests/*` should already be there).

## Stepping by hand

No engine frames run inside a synchronous loop, so call the per-frame functions yourself, in game order:

```gdscript
func step(delta: float) -> void:
	time += delta
	state._process(delta)          # game clock, habits, adaptation, combat mode
	spawner.tick(delta)
	hero._physics_process(delta)
	for e in main.get_tree().get_nodes_in_group("enemies"):
		e._physics_process(delta)
	_free_removed(main)            # queue_free never completes here: free by hand
```

Set the save path on the state node after `instantiate()` and before `add_child`, so the bots never touch the real save.

## Bot scripts

Each step is `{"do": "attack" | "dodge_left" | "dodge_right" | "wait", "wait": seconds}`, repeated in a loop. Build the timings from `Tuning` (for example `rhythm_max_gap * 0.5` for an in-rhythm press), so a rebalance never breaks the bots. Have at least one bot react like a person (dodge sideways when the nearest enemy winds up, alternating sides); a bot that stands still in contact dies and teaches you nothing.

## Running

```
godot --headless -s addons/gut/gut_cmdln.gd -gconfig=.gutconfig.json -gselect=test_bots
godot --headless res://tools/balance_report.tscn
```

A pass prints lines like `slime_block_heavy on after 17.9 s` and a report block per bot (`swordplay 5   1.7 min`, `counter ...  0.3 min`, `deaths  5`). Add the report as a CI step after the tests and paste the numbers into the PR.

## Traps already hit

- **Wait one physics frame before stepping.** Bodies added in the same frame are not in the physics space yet: `velocity` is set but `move_and_slide` moves nothing. In GUT `await wait_physics_frames(2)` in `before_each`; in the report `await get_tree().physics_frame` twice.
- **Run the report as a scene, not with `-s`.** In a `-s` SceneTree script, autoloads (`EventBus`) are not globals, so every game script fails to compile. A scene run (`godot --headless res://tools/x.tscn`) loads them; end with `get_tree().quit()`.
- Free `is_queued_for_deletion()` nodes and tick popups yourself, or dead enemies and floating text pile up.
- Read the first report for nonsense before balancing. Here every counter switched on 6 seconds into a fight: one sample made a habit share 1.0. The fix (a minimum sample count) was a real game bug a human only felt as "random".
