---
name: godot-rule-tests
description: Use when adding or changing pure game logic in a Godot 4 project (the rules/ folder: damage, drops, XP, loot tables, state machines) and you need a headless GUT unit test that runs locally and in CI, with a red-then-green check.
---

# Godot rule tests (GUT, headless)

Rules are plain GDScript (`extends RefCounted`, no nodes, no scene tree, no `Input`) under `res://rules/`. Test them with GUT. Keep engine-dependent code out of rules so the tests stay fast and deterministic.

## Naming

- Unit `rules/<unit>.gd` is tested by `tests/rules/test_<unit>.gd`.
- File extends `GutTest`; each test is `func test_<behavior>():`.
- Test names describe behavior (`test_crit_doubles_damage`), not implementation.

## Writing a test

```gdscript
extends GutTest

const Damage = preload("res://rules/damage.gd")

func test_armor_reduces_damage():
    assert_eq(Damage.apply(10, 3), 7)

func test_damage_never_negative():
    assert_eq(Damage.apply(2, 99), 0)
```

- Use `assert_eq`, `assert_almost_eq` (floats), `assert_true`, `assert_null`.
- Make randomness injectable: pass a seeded `RandomNumberGenerator` into the rule; never call global `randf()` inside a rule.
- Cover edges: zero, negative, max, empty list.

## Running

Config lives in `.gutconfig.json` (`dirs: ["res://tests/"]`, `include_subdirs: true`, `should_exit: true`).

```
godot --headless --import            # once after a fresh checkout (creates .godot/)
godot --headless -s addons/gut/gut_cmdln.gd -gconfig=.gutconfig.json
```

Non-zero exit code means failure. One file: add `-gselect=test_damage.gd`; one test: add `-gunit_test_name=test_armor_reduces_damage`.

CI runs the same command in the `test` job (Godot pinned to the exact version, installed as `godot` on PATH). Later jobs use `needs: test`.

If Godot cannot run locally (sandbox), push and read the Actions result for the job log; treat CI as the test runner.

## Red then green (required)

1. Write the test first; run it; confirm it fails for the expected reason (assertion message, not a parse error or missing file).
2. Implement the rule; run; confirm it passes.
3. To prove the suite can fail at all (new project or CI change), temporarily break one assertion, confirm the run goes red with a non-zero exit, then revert and confirm green.

A test never seen failing proves nothing.
