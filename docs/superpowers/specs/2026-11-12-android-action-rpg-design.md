# Android Action RPG — First Version Design

- **Date:** 2026-11-12
- **Status:** Draft, awaiting owner review
- **Working title:** "Adaptive Forest" (a placeholder. The real name is chosen before the Google Play listing.)

## 1. Purpose

- **Ship a real game on Google Play**, grown piece by piece through small updates.
- **Give the testing skills in this repo a real game to test.** Every testing method used on the game becomes a reusable skill in `video-game-test-skills` at the point it is first used.

### Roles and constraints

- The owner directs and play-tests. They have no coding experience and use only an Android phone.
- Claude writes all code, tests and build configuration.
- The game is Android only. There is no iOS version planned.
- The first version is offline and single-player, with no accounts, ads or in-app purchases.

## 2. The hook

> **You get stronger at whatever you do, and so do the monsters.**

- The player's skills grow from use, not from spending points.
- Skill milestones unlock perks.
- Enemies notice the player's habits, switch on counters to those habits, and announce it when they do.
- Changing fighting style makes enemies slowly forget.

This pushes the player to vary how they fight, and varied fighting is also what grows more skills and unlocks more perks.

## 3. First version: how it plays

### World and camera
- One hand-built, fixed forest area, a few screens wide.
- The home base camp sits at one edge.
- The view is top-down in landscape orientation, and the camera follows the hero.

### Controls (two thumbs)
- **Left thumb:** a floating virtual joystick that appears where the thumb lands in the left half of the screen. It controls movement.
- **Right thumb:** an **Attack** button and a **Dodge** button.

### Combat
| Move | Input | Effect |
|---|---|---|
| Combo A | Attack, Attack, Attack (in rhythm) | Slash, slash, **Heavy Finish** (a strong single-target hit) |
| Combo B | Attack, Attack, *short pause*, Attack | Slash, slash, **Spin Slash** (hits everything around the hero) |
| Dodge | Dodge | A roll in the movement direction, with brief invulnerability |
| Dodge Strike | Attack within a short window after a dodge | A quick counter-hit |

- **Perfect dodge:** a dodge that starts just before an enemy hit lands. Perks key off it.
- The combo rhythm windows are tunable values in content data.

### Player skills (kept forever)
| Skill | Grows when you… | Effect per level |
|---|---|---|
| Swordplay | land hits | more damage |
| Evasion | dodge enemy attacks (a perfect dodge counts extra) | faster roll, longer invulnerability |
| Toughness | take a hit and survive it | more max health |

- Each skill has levels driven by use. Growth per action and the level curve are content data.

### Perks (two per skill in the first version)
| Skill | Milestone | Perk |
|---|---|---|
| Swordplay | 5 | Dodge Strike deals bonus damage |
| Swordplay | 10 | Heavy Finish knocks enemies back |
| Evasion | 5 | A perfect dodge slows time briefly |
| Evasion | 10 | Dodging through an enemy damages it |
| Toughness | 5 | Healing potions heal more |
| Toughness | 10 | Once per life, survive a killing blow with 1 health |

### Adaptive enemies
- **Habit tracking:** the game keeps a rolling record of recent player actions:
  - share of combos that end in Heavy Finish compared with Spin Slash
  - dodge frequency
  - dodge direction (left or right of the attacker)
  - attack button spam rate
- **Counters:** each enemy type defines counters. Each counter names the habit it reacts to, a switch-on threshold and a switch-off (forget) threshold. Thresholds are content data.

| Enemy | Base behavior | Counter 1 | Counter 2 |
|---|---|---|---|
| Slime | Slow, high health, chases on sight | Overused Heavy Finish → blocks the third hit of Combo A | Overused Spin Slash → splits into two small slimes when hit by it |
| Wolf | Fast, lunges | Frequent dodging → waits out the dodge, then lunges | Dodges mostly to one side → circles toward the side you rarely dodge to |

- **Making adaptation visible:**
  - When a counter switches on, a message appears, for example *"The wolves have learned your dodge"*.
  - The enemy shows a visible tell, such as a different stance, color accent or animation.
  - A message also appears when a counter is forgotten.
- **Forgetting:** when the habit falls below the forget threshold, the counter switches off gradually, not instantly.
- **Persistence:** adaptation state is saved and **survives death**.
- **Respawning:** enemies respawn over time in fixed spawn zones.

### Items (lost on death unless stored)
- **Weapons:** dropped by enemies, each with a damage value. At least two are defined: Rusty Sword and Iron Sword. One weapon is equipped at a time.
- **Healing potions:** dropped by enemies. They are drunk from a quick-use button.
- **Bag:** a small, fixed number of slots for carried items. The size is content data.

### Home base camp
- **Storage chest:** items inside it are never lost.
- **Campfire:** resting there heals the hero fully and saves the game. Enemies cannot enter the camp.

### Death
- The hero respawns at camp.
- **Kept:** skills, perks, chest contents, enemy adaptation state.
- **Lost:** everything in the bag and the equipped weapon. A respawned hero with no weapon gets a basic starter stick, so they can always fight.

### Saving
- Saving is automatic. It happens on resting at the campfire, on death, and when the app is backgrounded.
- Save files carry a **schema version**, and older saves are migrated forward on load.

### Not in the first version
These are planned future updates, listed in a likely order. The order is not binding.
1. Real pixel-art graphics replace the placeholders.
2. **The killer keeps your stuff:** the enemy that kills you carries your dropped items. Defeat it to recover them, and they are lost for good if you die again first.
3. Sound and music.
4. Gathering resources.
5. Crafting at camp.
6. More regions and enemy types, each with new counters, plus more perks.
7. Quests and characters to talk to.
8. Bosses.

## 4. Architecture

### Engine
- **Godot 4**, using GDScript. The exact stable version is pinned in the build configuration.
- The web export uses Godot's single-threaded web build, so it runs on plain static hosting.

### Separation of rules and presentation
- **`rules/`:** pure game logic with no nodes that draw anything.
  - skills and perks
  - habit tracker and adaptation
  - inventory, chest and death rules
  - save and load, with migrations

  This code is testable headless in seconds.
- **`content/`:** data files (Godot Resources) for enemies, counters, weapons, potions, perks, skill curves and tunable numbers. Adding or rebalancing content means editing data, not code.
- **`game/`:** scenes and scripts for the world, hero, enemies, camera, input, HUD and messages. These read from `rules/` and `content/`.
- **`tests/`:** automated tests (see section 6).

### Communication between systems
- Systems communicate through a central **event bus** with named events, for example `player_hit_landed`, `player_dodged`, `perfect_dodge`, `enemy_defeated`, `player_died`, `counter_activated` and `counter_forgotten`.
- Systems never call into each other's internals.
- New systems, such as crafting, subscribe to existing events.

### Systems in the first version
- combat
- skills
- perks
- habit tracking and adaptation
- inventory and chest
- world and spawning
- save and load
- HUD and messages

## 5. Build and delivery

### Repositories
- **The game:** a new **public** GitHub repo, created by the owner (Claude cannot create repos). This design doc is copied there when it is created.
- **The testing skills:** this repo, `video-game-test-skills`.

### Pipeline (GitHub Actions, on every push to the game repo's default branch)
1. Run the rule tests and robot-player tests headless. Stop on any failure.
2. Export the **web build** and deploy it to **GitHub Pages**, so it can be played from a fixed link in a phone browser in landscape.
3. Export a **debug-signed APK** and attach it to a rolling "latest test build" GitHub Release.

### Later, for Google Play
- A release signing key will be stored as a GitHub Actions secret, and an Android App Bundle will be signed in CI. This is out of scope for the first version.

### Play-test loop
1. Claude announces what changed, with a short checklist.
2. The owner plays via the web link or the APK.
3. The owner reports back in chat, or as a GitHub issue.

## 6. Testing

1. **Rule tests:** unit tests on `rules/`, run headless with the GUT (Godot Unit Test) framework. Examples:
   - skill growth from actions
   - perk unlocks at milestones
   - counter switch-on and forget thresholds
   - adaptation persisting through death
   - bag lost and chest kept on death
   - save migration from an older schema
2. **Robot players:** headless simulations with scripted inputs. Examples:
   - a Heavy-Finish-only bot causes slimes to block
   - a varied bot causes counters to be forgotten
   - balance reports, such as time to reach Evasion 5
3. **Screenshot checks:** Claude loads the web build in headless Chromium, runs a scripted input sequence, captures screenshots at key moments and reviews them before announcing a build.
4. **Human play-test:** the owner, using the per-version checklist. Fun, feel and fairness are judged only here.

**Testing skills:** each method above is written as a skill in this repo the first time it is used on the game.

## 7. Art

- **First version:** simple, readable placeholder shapes, with hit flashes, Godot 2D lighting and particles. This proves combat and adaptation are fun before investing in art.
- **Target style:** **pixel art**, from packs whose license allows commercial use (for example CC0, or paid itch.io packs). Each pack's license is checked and recorded in the repo.
- Swapping art must not touch `rules/`.

## 8. Success criteria for the first version

- The game installs from the GitHub Release APK and plays from the web link on the owner's phone.
- A full session works: leave camp, fight slimes and wolves with both combos, dodge and Dodge Strike, gain skill levels, unlock at least one perk, pick up and store items, die, respawn with skills, chest and adaptation intact and the bag empty.
- At least one counter per enemy type switches on from a habit, shows its message and tell, and is forgotten after a style change.
- An old save loads after an update that changes the save schema (proven by a test).
- All automated tests pass in CI on every push.
- The owner's play-test verdict on the core question: *"Is adapting to adaptive enemies fun?"*

## 9. Risks

| Risk | Mitigation |
|---|---|
| Adaptation feels unfair or invisible | Clear messages and tells, gradual forgetting, tunable thresholds, and an explicit play-test checklist question |
| Touch controls feel bad for combos | Generous rhythm windows in content data, tuned from play-test feedback |
| Web build performs poorly on the phone | The APK is the reference for performance, and the web build is used for quick checks |
| Scope creep | Anything not in section 3 waits for a later update |
