# Android Action RPG — First Version Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first playable version of the adaptive-enemy action RPG described in the spec, delivered to the owner's phone as a web link and an APK on every push.

**Architecture:**
- Pure GDScript game logic lives in `rules/`. It has no drawing nodes and is unit-tested headless with GUT.
- Data files (Godot Resources) live in `content/`.
- Scenes live in `game/` and talk to the rules through an `EventBus` autoload.
- GitHub Actions runs the tests, then exports the web build to GitHub Pages and a debug APK to a rolling release.

**Tech Stack:**
- Godot **4.5.1-stable** (GDScript)
- GUT (Godot Unit Test), the 9.x release that supports Godot 4.5
- GitHub Actions, GitHub Pages
- Playwright with the pre-installed Chromium, for screenshot checks

**Spec:** `docs/superpowers/specs/2026-11-12-android-action-rpg-design.md`. It is copied into the game repo in Task 0, and executors read both documents.

## Global Constraints

- Android only. Landscape. Offline, single-player. No accounts, ads or purchases.
- Godot version is pinned exactly to `4.5.1-stable` everywhere: CI downloads, export templates and `project.godot` features.
- The web export uses the single-threaded build (`variant/thread_support=false`).
- `rules/` must not reference any `Node2D`, `Control` or scene. Allowed: `RefCounted`, `Resource`, and the `EventBus` autoload.
- All tunable numbers live in `content/` resources, never as literals in `game/` or `rules/`.
- Art in the first version is placeholder shapes only. Any future art pack's license is recorded in `ASSETS_LICENSES.md`.
- The game repo is public. Pages source is "GitHub Actions".
- Every new testing method used for the first time is written up as a skill in `video-game-test-skills/.claude/skills/<name>/SKILL.md` within the same task.
- The owner has no computer, so any manual owner step must be doable in the GitHub mobile app or the GitHub website on a phone.

## Review Focus

1. **App closed or backgrounded mid-fight:** progress since the last rest must be saved (`NOTIFICATION_APPLICATION_PAUSED`, and `NOTIFICATION_WM_CLOSE_REQUEST`). Test in Task 11.
2. **Missing or corrupt save file:** start a fresh game, never crash, and keep the bad file as `save.bak`. Test in Task 7.
3. **Picking up an item with a full bag:** the item stays on the ground and a "Bag full" message shows. Test in Task 6 (rule) and Task 11 (scene).
4. **Two thumbs at once:** the joystick and the buttons track separate touch indices, so moving while attacking works. Test in Task 9.
5. **A habit hovering right at a threshold:** the counter must not flicker on and off or spam messages. The on and off thresholds provide hysteresis. Test in Task 5.

---

### Task 0: Owner creates the game repo (manual, on the phone)

**Files:** none (owner actions), then a copy of the spec and plan into the new repo.

- [ ] **Step 1: Owner creates a public repo.** The working name is `adaptive-forest`, and the owner may choose another. Tick "Add a README". This can be done in the GitHub app (**+ → New repository**) or on github.com.
- [ ] **Step 2: Owner sets Settings → Pages → Source → "GitHub Actions".**
- [ ] **Step 3: Owner sets Settings → Actions → General → Workflow permissions → "Read and write".** The release upload needs this.
- [ ] **Step 4: Claude attaches the repo** with `add_repo` (access `push`) and clones it.
- [ ] **Step 5: Copy the spec and this plan** into the game repo at the same paths, commit them as `docs: add design spec and v1 plan`, and push.

---

### Task 1: Godot project, GUT, CI test job, and the first skill

**Files:**
- Create:
  - `project.godot`
  - `.gitignore` (`.godot/`, `*.import` cache, `build/`)
  - `addons/gut/` (vendored GUT at a pinned release tag, license kept)
  - `.gutconfig.json` (dirs: `res://tests/`, include subdirs, `should_exit: true`)
  - `tests/test_smoke.gd`
  - `.github/workflows/ci.yml`
- Skills repo: `.claude/skills/godot-rule-tests/SKILL.md`

**Interfaces:**
- Produces: CI job `test`, which later jobs depend on with `needs: test`.
- Produces: the test command `godot --headless -s addons/gut/gut_cmdln.gd -gconfig=.gutconfig.json`, where a non-zero exit means failure.

- [ ] **Step 1: Write `tests/test_smoke.gd`** with `extends GutTest` and `func test_engine_version(): assert_eq(Engine.get_version_info().minor, 5)`.
- [ ] **Step 2: Write `ci.yml` with the `test` job.**
  - Runs on `ubuntu-latest` and pushes to the default branch, plus PRs.
  - Downloads `Godot_v4.5.1-stable_linux.x86_64.zip` from the official GitHub release.
  - Runs `--headless --import` once, then the test command.
  - Each action is pinned by commit SHA.
- [ ] **Step 3: Push. CI must show `test` green** with 1 passing test. Then temporarily change the assertion to `4` and confirm CI goes red, then revert.
- [ ] **Step 4: Write the skill `godot-rule-tests`.** It covers how to write a GUT test for `rules/`, how to run it headless locally and in CI, the red-then-green check, and naming (`tests/rules/test_<unit>.gd`). Commit it to the skills repo.
- [ ] **Step 5: Commit** the game repo as `chore: godot project, GUT and CI test job`.

---

### Task 2: EventBus and content resource types

**Files:**
- Create:
  - `rules/event_bus.gd` (registered as autoload `EventBus`)
  - `content/types/skill_def.gd`
  - `content/types/perk_def.gd`
  - `content/types/counter_def.gd`
  - `content/types/enemy_def.gd`
  - `content/types/item_def.gd`
  - `content/types/tuning.gd`
- Create data:
  - `content/skills/{swordplay,evasion,toughness}.tres`
  - `content/perks/*.tres` (6 perks)
  - `content/enemies/{slime,wolf}.tres`
  - `content/counters/*.tres` (4 counters)
  - `content/items/{starter_stick,rusty_sword,iron_sword,healing_potion}.tres`
  - `content/tuning.tres`
- Test: `tests/rules/test_content.gd`

**Interfaces (all `class_name`, `extends Resource`, `@export` fields):**
- `SkillDef`:
  - `id: StringName`
  - `gain: Dictionary` (action StringName → float)
  - `level_thresholds: PackedFloat32Array` (cumulative points for levels 1..N)
  - `effect_per_level: float`
- `PerkDef`: `id: StringName`, `skill_id: StringName`, `milestone: int`, `title: String`
- `CounterDef`:
  - `id: StringName`, `enemy_id: StringName`, `habit: StringName`
  - `on_threshold: float`, `off_threshold: float`
  - `forget_rate: float` (strength lost per second)
  - `message_on: String`, `message_off: String`
- `EnemyDef`:
  - `id: StringName`, `max_health: int`, `speed: float`, `contact_damage: int`
  - `counters: Array[CounterDef]`
  - `drops: Dictionary` (ItemDef path → chance 0..1)
- `ItemDef`:
  - `id: StringName`, `display_name: String`
  - `kind: int` (enum `Kind {WEAPON, POTION}`)
  - `damage: int`, `heal: int`
- `Tuning`:
  - `rhythm_max_gap := 0.35`, `pause_max_gap := 0.8`, `dodge_strike_window := 0.4`
  - `perfect_dodge_window := 0.15`, `habit_window := 20`, `bag_size := 6`
  - `respawn_seconds := 30.0`
  - `adapt_ramp_rate := 0.5` (strength gained per second while a counter is active)
- `EventBus` signals:
  - `player_hit_landed(damage: int)`
  - `player_dodged(perfect: bool)`
  - `player_survived_hit(damage: int)`
  - `combo_finished(finisher: StringName)`
  - `enemy_attacked`
  - `enemy_defeated(enemy_id: StringName, position: Vector2)`
  - `player_died`
  - `counter_activated(counter: CounterDef)`
  - `counter_forgotten(counter: CounterDef)`
  - `perk_unlocked(perk: PerkDef)`
  - `message(text: String)`
  - `save_requested`

**Starting data values** (all tunable later):
- **Perks** (milestones and titles are exactly the spec's table): `swordplay_5_dodge_strike_bonus`, `swordplay_10_heavy_knockback`, `evasion_5_slowmo`, `evasion_10_dodge_through`, `toughness_5_potion_boost`, `toughness_10_last_stand`.
- **Counters:**
  - `slime_block_heavy`: habit `heavy_finish_share`, on 0.7, off 0.4
  - `slime_split_spin`: habit `spin_share`, on 0.7, off 0.4
  - `wolf_wait_dodge`: habit `dodge_rate`, on 0.7, off 0.4
  - `wolf_circle_side`: habit `dodge_side_bias`, on 0.5, off 0.25
  - All four use `forget_rate` 0.1.
  - Messages follow the pattern *"The wolves have learned your dodge"* / *"The wolves have forgotten your dodge"*.
- **Items:** stick damage 2, rusty 4, iron 7. Potion heals 30.
- **Enemies:** slime health 40, speed 40, contact damage 8. Wolf health 25, speed 130, contact damage 10.

- [ ] **Step 1: Write `test_content.gd`.**
  - Loads every `.tres` under `content/` and asserts it is non-null.
  - Every perk's `skill_id` names an existing skill, and the milestones are {5, 10} per skill.
  - Every counter has `off_threshold < on_threshold`.
  - Every enemy has exactly 2 counters.
- [ ] **Step 2: Push. Expect FAIL** (missing types).
- [ ] **Step 3: Implement the types, the `EventBus` and the data files** with the values above.
- [ ] **Step 4: Push. Expect PASS.**
- [ ] **Step 5: Commit** as `feat: content types, data and event bus`.

---

### Task 3: Skills and perks

**Files:** Create `rules/skill_set.gd`. Test `tests/rules/test_skill_set.gd`.

**Interfaces:**
- Consumes: `SkillDef`, `PerkDef`, `EventBus.perk_unlocked`.
- Produces: `class_name SkillSet extends RefCounted`:
  - `_init(skills: Array[SkillDef], perks: Array[PerkDef])`
  - `add_action(action: StringName, amount := 1.0) -> Array[PerkDef]` (returns perks newly unlocked, and emits `perk_unlocked` for each)
  - `level(skill_id: StringName) -> int`
  - `points(skill_id: StringName) -> float`
  - `has_perk(perk_id: StringName) -> bool`
  - `effect(skill_id: StringName) -> float` (`level * effect_per_level`)
  - `to_dict() -> Dictionary`, `load_dict(d: Dictionary) -> void`
- Action names used by the game:
  - `&"hit_landed"` and `&"perfect_dodge"` (Swordplay)
  - `&"dodge"` and `&"perfect_dodge"` (Evasion, with perfect worth more via `gain`)
  - `&"survived_hit"` (Toughness)

- [ ] **Step 1: Write the tests.**
  - `test_hits_raise_swordplay`: enough `hit_landed` to cross threshold 1, so `level(&"swordplay") == 1`.
  - `test_perfect_dodge_counts_extra`: perfect dodge points are greater than plain dodge points.
  - `test_milestone_unlocks_perk_once`: reaching level 5 returns `[swordplay_5…]` once. More actions return `[]`.
  - `test_round_trip`: `to_dict` then `load_dict` keeps levels and perks.
- [ ] **Step 2: Push. Expect FAIL.**
- [ ] **Step 3: Implement `SkillSet`.**
- [ ] **Step 4: Push. Expect PASS.**
- [ ] **Step 5: Commit** as `feat: use-based skills and milestone perks`.

---

### Task 4: Habit tracker

**Files:** Create `rules/habit_tracker.gd`. Test `tests/rules/test_habit_tracker.gd`.

**Interfaces:**
- Produces: `class_name HabitTracker extends RefCounted`:
  - `_init(window: int)`
  - `record_combo(finisher: StringName)` (`&"heavy_finish"` or `&"spin_slash"`)
  - `record_enemy_attack(dodged: bool, side: int)` (side is −1 for left, +1 for right, 0 if not dodged)
  - `record_attack_press(gap_seconds: float)`
  - `value(habit: StringName) -> float` (range 0..1)
  - `favored_side() -> int`
  - `to_dict()`, `load_dict(d)`
- Habit definitions, each over the last `window` relevant events:
  - `heavy_finish_share` = heavy finishes ÷ combos
  - `spin_share` = spin slashes ÷ combos
  - `dodge_rate` = dodged ÷ enemy attacks
  - `dodge_side_bias` = `abs(left_share − 0.5) * 2` over dodges
  - `attack_spam` = share of presses with gap < 0.15s
  - Every value is 0 with no events.

- [ ] **Step 1: Write the tests.**
  - `test_heavy_only_gives_share_1`
  - `test_window_drops_old_events`: window 20, 20 heavy then 20 spin gives heavy share 0.
  - `test_dodge_side_bias_and_favored_side`: 15 left and 5 right gives bias 0.5 and favored −1.
  - `test_empty_is_zero`
  - `test_round_trip`
- [ ] **Step 2: Push. Expect FAIL.**
- [ ] **Step 3: Implement it** with ring buffers per event type.
- [ ] **Step 4: Push. Expect PASS.**
- [ ] **Step 5: Commit** as `feat: habit tracker`.

---

### Task 5: Adaptation state

**Files:** Create `rules/adaptation.gd`. Test `tests/rules/test_adaptation.gd`.

**Interfaces:**
- Consumes: `HabitTracker.value`, `CounterDef`, `Tuning.adapt_ramp_rate`, and `EventBus.counter_activated` / `counter_forgotten` / `message`.
- Produces: `class_name Adaptation extends RefCounted`:
  - `_init(counters: Array[CounterDef], ramp_rate: float)`
  - `update(tracker: HabitTracker, delta: float) -> void`
  - `strength(counter_id: StringName) -> float` (range 0..1)
  - `is_active(counter_id: StringName) -> bool`
  - `to_dict()`, `load_dict(d)`

**Rule:**
- When the habit is at or above `on_threshold` and the counter is inactive, it becomes active. It emits `counter_activated` and `message(message_on)`.
- While active and the habit is above `off_threshold`, strength rises by `ramp_rate * delta`, capped at 1.
- When the habit is below `off_threshold`, strength falls by `forget_rate * delta`.
- When strength reaches 0, the counter becomes inactive and emits `counter_forgotten` and `message(message_off)`.

- [ ] **Step 1: Write the tests.**
  - `test_activates_at_threshold_and_emits_once` (use `watch_signals(EventBus)`)
  - `test_strength_ramps_to_one`
  - `test_forgets_gradually`: after the habit drops, it is still active after 1s and inactive after `1/forget_rate` s, emitting once.
  - `test_no_flicker_between_thresholds`: the habit oscillating between 0.45 and 0.65 after activation emits nothing new (Review Focus 5).
  - `test_round_trip`
- [ ] **Step 2: Push. Expect FAIL.**
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Push. Expect PASS.**
- [ ] **Step 5: Commit** as `feat: enemy adaptation with gradual forgetting`.

---

### Task 6: Inventory, chest and death rules

**Files:** Create `rules/inventory.gd`. Test `tests/rules/test_inventory.gd`.

**Interfaces:**
- Produces: `class_name Inventory extends RefCounted`:
  - `_init(bag_size: int, starter: ItemDef)`
  - `bag: Array[ItemDef]`, `chest: Array[ItemDef]`, `equipped: ItemDef`
  - `pick_up(item) -> bool` (false when the bag is full)
  - `equip(index: int)`
  - `drink_potion(heal_bonus := 0) -> int` (returns the heal amount, or 0 if there is no potion)
  - `store(index: int)`, `take(index: int) -> bool`
  - `apply_death()`
  - `to_dict()`, `load_dict(d, items_by_id: Dictionary)`
- **Death rule:** empty the bag and set `equipped = starter`. The chest is untouched. The starter is never stored in the bag, so it never duplicates.

- [ ] **Step 1: Write the tests.**
  - `test_pick_up_until_full_then_false` (Review Focus 3)
  - `test_death_clears_bag_and_weapon_keeps_chest`
  - `test_death_twice_gives_single_starter`
  - `test_potion_boost_applies`
  - `test_round_trip`
- [ ] **Step 2: Push. Expect FAIL.**
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Push. Expect PASS.**
- [ ] **Step 5: Commit** as `feat: inventory, chest and death rule`.

---

### Task 7: Save and load with schema migrations

**Files:**
- Create `rules/save_game.gd`.
- Test `tests/rules/test_save_game.gd`, with fixtures in `tests/fixtures/`.

**Interfaces:**
- Consumes: `to_dict` / `load_dict` from `SkillSet`, `HabitTracker`, `Adaptation` and `Inventory`.
- Produces: `class_name SaveGame extends RefCounted`:
  - `const SCHEMA_VERSION := 1`
  - `static var migrations: Dictionary` (from_version int → Callable(Dictionary) → Dictionary)
  - `static func write(path: String, state: Dictionary) -> Error` (adds `"schema"`, and writes JSON atomically via a temp file and rename)
  - `static func read(path: String) -> Dictionary` (returns `{}` when the file is missing, and runs migrations up to `SCHEMA_VERSION`)
  - On a parse failure it renames the file to `save.bak` and returns `{}`.
- The default path is `user://save.json`.

- [ ] **Step 1: Write the tests.**
  - `test_round_trip`
  - `test_missing_file_returns_empty`
  - `test_corrupt_file_backed_up_and_empty` (Review Focus 2)
  - `test_migration_runs`: register a temporary migration 1→2 adding `"new_field"`, temporarily set the target to 2 via the test hook `SaveGame._target_override`, and read a v1 fixture. The result has `"new_field"` and `schema == 2`.
- [ ] **Step 2: Push. Expect FAIL.**
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Push. Expect PASS.**
- [ ] **Step 5: Commit** as `feat: versioned saves with migrations`.

---

### Task 8: Combo recognizer

**Files:** Create `rules/combo.gd`. Test `tests/rules/test_combo.gd`.

**Interfaces:**
- Consumes: `Tuning` windows, `EventBus.combo_finished`.
- Produces: `class_name Combo extends RefCounted`:
  - `_init(t: Tuning)`
  - `press_attack(now: float) -> StringName`, returning one of `&"slash1"`, `&"slash2"`, `&"heavy_finish"`, `&"spin_slash"` or `&"dodge_strike"`
  - `notify_dodge(now: float)`
  - `reset()`

**Rule:**
- A press within `dodge_strike_window` after a dodge gives `dodge_strike`, and the chain resets.
- Otherwise a gap over `pause_max_gap` (or the first press) gives `slash1`.
- The second press in the chain gives `slash2`.
- The third press with a gap ≤ `rhythm_max_gap` gives `heavy_finish`. A gap in (`rhythm_max_gap`, `pause_max_gap`] gives `spin_slash`.
- A finisher emits `combo_finished` and resets the chain.

- [ ] **Step 1: Write the tests.**
  - `test_rhythm_gives_heavy` (times 0, 0.2, 0.4)
  - `test_pause_gives_spin` (0, 0.2, 0.75)
  - `test_long_gap_restarts` (0, 0.2, 1.5 gives slash1)
  - `test_dodge_strike` (dodge 1.0, press 1.2)
  - `test_emits_combo_finished`
- [ ] **Step 2: Push. Expect FAIL.**
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Push. Expect PASS.**
- [ ] **Step 5: Commit** as `feat: combo recognizer`.

---

### Task 9: Hero, controls and the world scene

**Files:**
- Create:
  - `game/main.tscn` + `game/main.gd`
  - `game/world/forest.tscn` (TileMapLayer with placeholder tiles, collision, `SpawnZone` markers, camp area)
  - `game/hero/hero.tscn` + `hero.gd`
  - `game/ui/touch_controls.tscn` + `touch_controls.gd`
  - `game/input_map.gd`
- Test: `tests/game/test_hero.gd`, `tests/game/test_touch_controls.gd`

**Interfaces:**
- Consumes: `Combo`, `Tuning`, `EventBus`.
- Produces in `hero.gd`:
  - `func move_input(dir: Vector2)`
  - `func attack(now: float)`
  - `func dodge(now: float)`
  - signals `hit_enemy(enemy: Node, damage: int)` and `died`
  - fields `health: int`, `max_health: int`
- Produces in `touch_controls.gd`:
  - emits `move(dir: Vector2)`, `attack_pressed`, `dodge_pressed`, `potion_pressed`
  - the floating joystick is limited to the left half of the screen
  - each control remembers its own touch `index`
- Keyboard fallback, for desktop and the screenshot checks: WASD to move, J to attack, K to dodge, H for a potion.
- `project.godot`:
  - landscape orientation, 1280×720 base, stretch mode `canvas_items`, aspect `expand`
  - `input_devices/pointing/emulate_touch_from_mouse=true`

Placeholder visuals:
- Hero: a blue circle with a white sword arc.
- Each move has its own arc and flash color.
- Dodge: a short dash with translucency during invulnerability.

- [ ] **Step 1: Write the tests.**
  - `test_move_input_moves_hero` (simulate frames)
  - `test_dodge_grants_invulnerability_then_ends`
  - `test_three_rhythm_attacks_emit_heavy_finish`
  - `test_joystick_and_attack_with_two_touch_indices`: a touch at index 0 drags the joystick while a touch at index 1 presses Attack. Both register (Review Focus 4).
  - `test_buttons_inside_viewport_at_19_5x9_and_4x3`
- [ ] **Step 2: Push. Expect FAIL.**
- [ ] **Step 3: Implement the scenes and scripts.**
- [ ] **Step 4: Push. Expect PASS.**
- [ ] **Step 5: Commit** as `feat: hero, two-thumb controls and forest scene`.

---

### Task 10: Enemies, counters in action, and spawning

**Files:**
- Create:
  - `game/enemies/enemy.gd` (base)
  - `game/enemies/slime.tscn` + `slime.gd`
  - `game/enemies/wolf.tscn` + `wolf.gd`
  - `game/world/spawner.gd`
  - `game/pickups/pickup.tscn` + `pickup.gd`
- Test: `tests/game/test_enemies.gd`

**Interfaces:**
- Consumes: `EnemyDef`, `Adaptation.strength/is_active`, `EventBus`.
- Produces in `enemy.gd`:
  - `@export var def: EnemyDef`
  - `func take_hit(damage: int, move: StringName) -> void`
  - `var adaptation: Adaptation` (injected by `main.gd`)
- **Counter behaviors.** Each applies with probability equal to `strength(counter_id)` when relevant:
  - `slime_block_heavy`: a `heavy_finish` hit is blocked and deals 0 damage.
  - `slime_split_spin`: a `spin_slash` hit spawns 2 slimes at half health.
  - `wolf_wait_dodge`: the wolf feints, waits until the hero's dodge ends, then lunges.
  - `wolf_circle_side`: the wolf approaches from `−favored_side`.
- **Tell:** an active counter tints the enemy with an orange outline and shows a small "!" icon.
- Enemies emit `enemy_attacked` when they start an attack, and `enemy_defeated` on death.
- **Perfect dodge:** when an enemy's hit would have landed within `Tuning.perfect_dodge_window` after the hero's dodge started, the hero emits `player_dodged(true)`. Any other dodge of an incoming attack emits `player_dodged(false)`. When a hit lands and the hero survives it, the hero emits `player_survived_hit`.
- **Drops** roll from `def.drops`. The spawner refills each `SpawnZone` after `Tuning.respawn_seconds`. No spawns happen inside the camp area.

- [ ] **Step 1: Write the tests.**
  - `test_slime_blocks_heavy_when_counter_full` (strength forced to 1)
  - `test_slime_takes_heavy_when_inactive`
  - `test_spin_splits_slime_when_active`
  - `test_wolf_waits_for_dodge_end`
  - `test_wolf_circles_unfavored_side`
  - `test_no_spawn_in_camp`
  - `test_defeat_emits_and_drops`
  - `test_perfect_dodge_detected_inside_window_only`
- [ ] **Step 2: Push. Expect FAIL.**
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Push. Expect PASS.**
- [ ] **Step 5: Commit** as `feat: slime and wolf with adaptive counters and spawning`.

---

### Task 11: Camp, HUD, messages, death flow and saving triggers

**Files:**
- Create:
  - `game/world/camp.tscn` (chest, campfire, enemy-blocking area)
  - `game/ui/hud.tscn` + `hud.gd` (health bar, skill levels, equipped weapon, bag button)
  - `game/ui/message_banner.gd`
  - `game/ui/bag_panel.tscn` + `bag_panel.gd`
  - `game/ui/chest_panel.tscn` + `chest_panel.gd`
  - `game/game_state.gd`
- Modify: `game/main.gd`
- Test: `tests/game/test_game_flow.gd`

**Interfaces:**
- Consumes: everything from Tasks 3–10.
- Produces: `class_name GameState extends Node`. It owns:
  - `skills: SkillSet`, `habits: HabitTracker`, `adaptation: Adaptation`, `inventory: Inventory`
  - `func save() -> void`, `func load() -> void`
  - It wires `EventBus` signals to the rules: hit → skills, combo → habits, and so on.
  - It calls `adaptation.update(habits, delta)` in `_process`.
- Skill effects are read via `skills.effect(id)` at the point of use:
  - Swordplay adds damage to every hit.
  - Evasion makes the roll faster and the invulnerability longer.
  - Toughness adds max health.
- Perk effects are read via `skills.has_perk` at the point of use in hero, enemy and inventory code:
  - slow-motion via `Engine.time_scale` for 0.5s
  - knockback impulse
  - dodge-through damage
  - potion bonus
  - last stand once per life
- Messages queue and show one at a time, 2.5s each.
- **Saving happens on:**
  - resting at the campfire, which also heals fully
  - `player_died`
  - `NOTIFICATION_APPLICATION_PAUSED`
  - `NOTIFICATION_WM_CLOSE_REQUEST`

- [ ] **Step 1: Write the tests.**
  - `test_full_death_cycle`: skills, perks, chest and adaptation persist. The bag is empty and the starter is equipped. The hero is at camp with full health.
  - `test_rest_heals_and_saves`
  - `test_pause_notification_saves` (Review Focus 1)
  - `test_full_bag_leaves_pickup_and_messages` (Review Focus 3)
  - `test_counter_message_shown`
  - `test_last_stand_once_per_life`
- [ ] **Step 2: Push. Expect FAIL.**
- [ ] **Step 3: Implement it.**
- [ ] **Step 4: Push. Expect PASS.**
- [ ] **Step 5: Commit** as `feat: camp, HUD, death flow and auto-save`.

---

### Task 12: Web build to Pages, debug APK to a release, and screenshot checks

**Files:**
- Create: `export_presets.cfg` (presets `Web` and `Android`)
- Modify: `.github/workflows/ci.yml`
- Create: `tools/screenshot_check.mjs`
- Skills repo: `.claude/skills/godot-ci-delivery/SKILL.md`, `.claude/skills/web-screenshot-check/SKILL.md`

**Interfaces:**
- **Job `web`** (`needs: test`, default branch only):
  - Downloads the 4.5.1 export templates.
  - Runs `godot --headless --export-release Web build/web/index.html`.
  - Uploads and deploys with `actions/upload-pages-artifact` and `actions/deploy-pages`.
- **Job `android`** (`needs: test`):
  - Sets up JDK 17 and the Android SDK.
  - Generates a debug keystore in the job.
  - Runs `--export-debug Android build/game-debug.apk`.
  - Runs `gh release upload test-build build/game-debug.apk --clobber`, creating the `test-build` prerelease if it is missing.
- **`screenshot_check.mjs <url> <outdir>`** runs locally here with the pre-installed Playwright:
  - viewport 1280×720
  - waits for the canvas and the first frame
  - sends a keyboard script: move, J×3, J J pause J, K then J
  - saves `01-start.png` through `05-dodge-strike.png`
  - exits non-zero if the canvas never appears or console errors are logged

- [ ] **Step 1: Add the export presets and both jobs. Push.** Expect CI green, a Pages URL in the `web` job summary, and `game-debug.apk` on the `test-build` release.
- [ ] **Step 2: Write `screenshot_check.mjs` and run it against the Pages URL.** Expect exit 0 and 5 screenshots. Review each screenshot for the hero, enemies and move arcs.
- [ ] **Step 3: Write the skills `godot-ci-delivery` and `web-screenshot-check`.** Commit them to the skills repo.
- [ ] **Step 4: Commit** the game repo as `ci: web build to Pages, debug APK release, screenshot check`.
- [ ] **Step 5: Owner checkpoint.** Send the owner the Pages link and the release link, with the first play-test checklist (Task 14 format). The owner confirms both work on their phone.

---

### Task 13: Robot players and balance report

**Files:**
- Create:
  - `tests/sim/bot.gd` (scripted input driver for `main.tscn`, run headless)
  - `tests/sim/test_bots.gd`
  - `tools/balance_report.gd`
- Skills repo: `.claude/skills/godot-robot-players/SKILL.md`

**Interfaces:**
- Consumes: `hero.move_input/attack/dodge`, `GameState`.
- Produces: `class_name Bot extends RefCounted`:
  - `_init(main: Node, script: Array[Dictionary])`
  - `step(delta: float)`
- Script steps look like `{ "do": "attack", "at": 0.2 }` and `{ "do": "seek_nearest" }`.
- `balance_report.gd` is run with `godot --headless -s tools/balance_report.gd`. It prints the simulated minutes to Swordplay 5, Evasion 5 and the first counter activation.

- [ ] **Step 1: Write the tests.**
  - `test_heavy_only_bot_triggers_slime_block` (within 10 simulated minutes)
  - `test_varied_bot_makes_counter_forget`
  - `test_bot_survives_first_minute_at_camp_edge`
- [ ] **Step 2: Push. Expect FAIL** (no `Bot` yet).
- [ ] **Step 3: Implement `Bot` and the report.**
- [ ] **Step 4: Push.** Expect PASS, with the report printed in the CI log. Paste the numbers into the PR.
- [ ] **Step 5: Write the skill `godot-robot-players`.** Commit both repos (game: `test: robot players and balance report`).

---

### Task 14: Play-test checklist and the first-version sign-off

**Files:**
- Create in the game repo: `docs/playtests/v1-checklist.md`
- Skills repo: `.claude/skills/playtest-checklist-report/SKILL.md`

- [ ] **Step 1: Write `v1-checklist.md`.** Use plain language, one line per check, covering each success criterion in spec §8. It must include the core question *"Is adapting to adaptive enemies fun?"*, plus *"Did you notice when enemies adapted?"* and *"Did it feel fair?"*.
- [ ] **Step 2: Owner plays and answers in chat.** Claude records the answers as `docs/playtests/v1-report.md` and opens issues for each problem found.
- [ ] **Step 3: Write the skill `playtest-checklist-report`.** It covers how to write a checklist from a spec and turn owner answers into a report and issues. Commit it to the skills repo.
- [ ] **Step 4: Verify.** CI is green, all spec §8 criteria are checked off in the report (or have an open issue), and the release and Pages builds match the last commit.
- [ ] **Step 5: Commit** as `docs: v1 play-test checklist and report`.
