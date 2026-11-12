---
name: playtest-report
description: Use when the owner reports back after play-testing a game build ("the wolves felt too fast", "the button was too small") and the notes need turning into a clear report and a list of issues to fix.
---

# Play-test report

Turn loose, casual notes into a report the next session can act on. The owner plays on a phone and writes in plain language. Keep your replies short and plain, with no jargon.

## 1. Collect the facts

Ask for anything missing, one short question at a time:
- Which build: the web link or the APK, and roughly when it was published.
- The phone, and whether it was held sideways.
- The three core questions, if not already answered: *Was it fun? Did you notice when enemies adapted to you? Did it feel fair?*

## 2. Write the report

Save it in the game repo as `docs/playtests/<version>-report.md`:

```markdown
# Play-test <version>: <date>

**Build:** <link or commit>   **Device:** <phone>

## Verdict
<two sentences: fun or not, and the main reason>

## Core questions
- Fun? ...
- Noticed adaptation? ...
- Fair? ...

## What worked
- ...

## Problems
| # | What happened | Where | Severity | Likely cause or fix |
|---|---|---|---|---|

## Next steps
1. ...
```

Rules:
- Quote the owner's own words for each problem.
- Severity: **Blocker** (can't play), **Bad** (plays wrong), **Annoying** (works but feels off), **Idea** (wish list).
- Never guess a cause as fact. Write "likely" and say what to check.
- Check the build's success criteria in the game's design spec and mark each one passed, failed or not tried.

## 3. Offer issues, don't file them

List the problems and ask whether to open one GitHub issue per **Blocker** and **Bad** item. Filing issues notifies people and is hard to undo, so wait for a yes. Group **Annoying** and **Idea** items into one issue.

## 4. Hand over

End with the three most valuable things to fix next, and offer to start on the first one.
