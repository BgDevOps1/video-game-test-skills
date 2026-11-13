# video-game-test-skills

A library of Claude Code skills for testing video games, written as the testing methods are first used on a real game.

The first game is **Adaptive Forest**, in the `adaptive-forest` repo (Godot 4.5.1, Android only). Each testing method used there becomes a reusable skill here, so it can be applied to future games.

The owner has no coding background and only a phone. Keep chat replies short and in plain language.

## Layout

```
.claude/skills/<name>/SKILL.md   one folder per skill
.claude/settings.json            plugin settings for this repo
docs/superpowers/                design spec and implementation plan for the first game
```

## Skills so far

| Skill | Use it for |
|---|---|
| `godot-rule-tests` | Headless GUT unit tests for pure game logic in `rules/`, with a red-then-green check |
| `godot-scene-tests` | Headless GUT tests for scenes and node scripts: physics stepping, multi-touch, layout at several aspect ratios, injected game time |
| `godot-robot-players` | Scripted bots play the real main scene, stepped by hand, for whole-session tests and a balance report |
| `web-screenshot-check` | Build the web export locally, screenshot it in headless Chromium at phone size, and look for missing or off-screen UI |

More are planned as each method is first used: CI delivery of web and APK builds, play-test checklists.

## Writing a skill

- One folder per skill: `.claude/skills/<name>/SKILL.md`. The name is lowercase with hyphens.
- Start with YAML frontmatter holding only `name` and `description`.
- The `description` starts with "Use when ..." and names the situation, not the contents. Claude reads it to decide whether to load the skill.
- Keep the body short and practical: the commands to run, the file naming, one small example, and the traps already hit. Don't repeat what Claude already knows.
- Only write down a method that has actually been run on a real game. Include the exact command and what a pass looks like.
- Match the house style of `godot-rule-tests`.

## Working here

- Work on a feature branch. The owner merges pull requests from the GitHub app.
- Don't add a game-specific rule to a skill. Game-specific rules belong in that game's own `CLAUDE.md`.
