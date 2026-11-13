---
name: web-screenshot-check
description: Use when a Godot 4 game has a web export and you need to actually see it run (layout, HUD, buttons, art) from a cloud session, before or after the owner play-tests. Builds the web export locally, serves it on localhost and screenshots it in headless Chromium at phone size.
---

# Web screenshot check (Godot 4 web export + headless Chromium)

Unit and scene tests cannot see a HUD hanging off the screen. A screenshot can. Run this after any change to `game/ui/` or layout, and once before handing a build to the owner.

## Why build locally

The live GitHub Pages site is usually blocked by the cloud container's network policy (`ERR_TUNNEL_CONNECTION_FAILED`, proxy status shows `403 to CONNECT`). `github.com` release downloads are allowed, so build the web export yourself.

## One-time: export templates (web only)

The full `.tpz` is about 1.3 GB and the connection may reset part-way. Resume with `-C -`:

```
mkdir -p $SCRATCH/tpl && cd $SCRATCH/tpl
for i in 1 2 3; do curl -fsSL -C - -o t.tpz \
  https://github.com/godotengine/godot/releases/download/4.5.1-stable/Godot_v4.5.1-stable_export_templates.tpz && break; done
unzip -tq t.tpz                      # "No errors detected"
unzip -q t.tpz 'templates/web_*' templates/version.txt
d=$HOME/.local/share/godot/export_templates/4.5.1.stable
mkdir -p $d && mv templates/* $d/
```

Match the version to the project's pinned Godot (dir name is `4.5.1.stable`). The project needs a `Web` preset in `export_presets.cfg`.

## Export, serve, screenshot

```
godot --headless --import            # first time only
godot --headless --export-release Web $SCRATCH/web/index.html
(cd $SCRATCH/web && python3 -m http.server 8123 &)
node shot.js
```

`shot.js` (Playwright and Chromium are pre-installed; `require('playwright')` works anywhere):

```js
const { chromium } = require('playwright');
(async () => {
  const b = await chromium.launch({ args: ['--use-gl=swiftshader','--enable-unsafe-swiftshader','--ignore-gpu-blocklist'] });
  const p = await b.newPage({ viewport: { width: 844, height: 390 }, deviceScaleFactor: 1 });
  p.on('console', m => console.log('console:', m.text().slice(0,200)));
  p.on('pageerror', e => console.log('pageerror:', e.message.slice(0,200)));
  await p.goto('http://localhost:8123/index.html', { waitUntil: 'load', timeout: 60000 });
  await p.waitForTimeout(15000);
  await p.screenshot({ path: process.argv[2] || 'web.png' });
  await b.close();
})();
```

Then open the PNG with the Read tool and look at it.

## What a pass looks like

- Console shows `Godot Engine v4.5.1`, `WebGL 2.0`, and no `pageerror`. "GPU stall due to ReadPixels" lines are harmless.
- The picture shows the world, the hero, the HUD text and bars, every touch button and every menu button, all inside the frame.
- Check each widget you expect by name; a missing one is a bug until proven otherwise.

Use 844x390 (landscape phone). Also try 360x800 portrait if the game supports it. Use `deviceScaleFactor: 1` for your own looks.

## Traps already hit

- **Anchors without offsets.** `set_anchors_preset(PRESET_FULL_RECT)` on a freshly created `Control` leaves it zero-width, so anything anchored to the right edge sits off screen at negative x. Use `set_anchors_and_offsets_preset`. The Bag/Chest/Rest buttons were invisible for this reason and only the screenshot showed it.
- Pair the fix with a test: put the UI in a `Control` sized like a phone, `await wait_frames(2)`, and assert each button's `get_global_rect()` is inside the screen rect. Prove it red first.
- Export the same project folder you changed; a stale `index.pck` shows the old build.
- The default single-threaded web build runs from plain `http.server`; no special headers needed.
